header
volley不支持直接添加header, 需要继承Request类重写getHeaders()方法, getHeaders这个方法名也非常让人误解
okhttp支持直接在Request.Builder中通过addHeader()直接添加header

multipart

cache
cookie
