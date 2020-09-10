---
layout: post
title:  "ImageSpan导致的内存泄漏"
date:   2020-09-09 18:21:26 +0800
categories: android
---


leakCanary报了一个内存泄漏, 当activity旋转后, 新的EditText竟然保持了老的activity的引用

路径是:
  * 新的EditText对象
  * Charsequence mLastValueSentToAutofillManager    // 实际是一个SpannableStringBuilder
  * Object[] mSpans
  * ImageSpan
  * Context // 老的Activity

这里使用了ImageSpan, 是为了实现自定义的表情功能, 而这个context引用是从ImageSpan的构造方法中传进去的

    public ImageSpan(@NonNull Context context, @DrawableRes int resourceId,
            int verticalAlignment) {
        super(verticalAlignment);
        mContext = context;
        mResourceId = resourceId;
    }

如果查看这个mContext在ImageSpan中用到的地方, 会发现唯一的用处是在getDrawable方法中调用了Context.getDrawable

    @Override
    public Drawable getDrawable() {
        Drawable drawable = null;

        if (mDrawable != null) {
            drawable = mDrawable;
        } else if (mContentUri != null) {
            ...
        } else {
            try {
                drawable = mContext.getDrawable(mResourceId);
                drawable.setBounds(0, 0, drawable.getIntrinsicWidth(),
                        drawable.getIntrinsicHeight());
            } catch (Exception e) {
                
            }
        }

        return drawable;
    }

所以解决这个问题的方法很简单, 有两个选项
1. 传入Application Context
2. 直接获取drawable, 使用另一个不用context的构造函数:

        public ImageSpan(@NonNull Drawable drawable, int verticalAlignment) {
            super(verticalAlignment);
            mDrawable = drawable;
        }

其中第二个方法毫无疑问是可以的, 而第一个方法使用applicationContext, 行不行取决于drawable资源是否使用了当前activity主题中的属性

到这里问题已经解决, 再次感叹一下Android这种随时随地重建activity到设计巨坑无比

but, 还是来研究下新的EditText是如何持有老的Activity的引用的吧

查看TextView的代码后发现  
实际上这个mLastValueSentToAutofillManager和EditText.mText是同一个对象  
在setText方法的最后会直接或间接地调用notifyAutoFillManagerAfterTextChangedIfNeeded这个方法, 通知系统的自动填充服务  
而这个方法的逻辑上大约是:

    private void notifyAutoFillManagerAfterTextChangedIfNeeded() {
        ...
        final AutofillManager afm = mContext.getSystemService(AutofillManager.class);
        ...
        if (mLastValueSentToAutofillManager == null || !mLastValueSentToAutofillManager.equals(mText)) {
            afm.notifyValueChanged(TextView.this);
            mLastValueSentToAutofillManager = mText;
        }
        ...
    }

所以这个mLastValueSentToAutofillManager只是mText的一个缓存值

而mText之所以指向了老的activity, 肯定是因为View的save/restore机制保留了之前的mText对象

查看一下TextView.onSaveInstanceState和onRestoreInstanceState方法

    SavedState ss = new SavedState(superState);

    if (freezesText) {
        if (mText instanceof Spanned) {
            final Spannable sp = new SpannableStringBuilder(mText);
            ...
            ss.text = sp;
        } else {
            ss.text = mText.toString();
        }
    }

    public void onRestoreInstanceState(Parcelable state) {
        SavedState ss = (SavedState) state;
        super.onRestoreInstanceState(ss.getSuperState());

        // XXX restore buffer type too, as well as lots of other stuff
        if (ss.text != null) {
            setText(ss.text);
        }
    }

1. getFreezesText对于EditText是总是true的
2. SavedState是TextView的内部类, 继承了View.BaseSavedState, 这里text成员基本上是直接保存了mText对象
3. onRestore中又把ss.text直接设置到了新的EditText上


一个奇怪的问题是, 这里所有涉及保存状态的类都是实现了Parcelable的, 而间接持有Activity对象的spannable又怎么可能parcel呢  
查看一下TextView.SavedState这个类的打包方法:

    public void writeToParcel(Parcel out, int flags) {
            super.writeToParcel(out, flags);
            out.writeInt(selStart);
            out.writeInt(selEnd);
            out.writeInt(frozenWithFocus ? 1 : 0);
            TextUtils.writeToParcel(text, out, flags);
            ...

text这个CharSequence的打包流程是自定义的, 封装在了TextUtils中(用了这么久的TextUtils从没发现这个方法)

    public static void writeToParcel(CharSequence cs, Parcel p, int parcelableFlags) {
        if (cs instanceof Spanned) {
            p.writeInt(0);
            p.writeString(cs.toString());

            Spanned sp = (Spanned) cs;
            Object[] os = sp.getSpans(0, cs.length(), Object.class);

            for (int i = 0; i < os.length; i++) {
                Object o = os[i];
                Object prop = os[i];

                if (prop instanceof CharacterStyle) {
                    prop = ((CharacterStyle) prop).getUnderlying();
                }

                if (prop instanceof ParcelableSpan) {
                    ...
                }
            }

            p.writeInt(0);
        } else {
            p.writeInt(1);
            ...
        }
    }

首先这个方法对cs是否是spanned进行了区分, 如果把ImageSpan代入for循环的o对对象中, 会发现它虽然是CharacterStyle, 但是它不是ParcelableSpan, 因此搞了半天它是没有被打包的

这说明: 
1. 旋转屏幕的save/restore流程还没走打包那一步, 恢复时用的是当前虚拟机中的ActivityRecord.state对象
2. 如果app在后台被系统干掉, 恢复的时候, ImageSpan默认是不会恢复的

不过在实现支持表情的EditText的时候, 如果是在afterTextChanged检查是否需要转成表情, 就覆盖了2的问题了