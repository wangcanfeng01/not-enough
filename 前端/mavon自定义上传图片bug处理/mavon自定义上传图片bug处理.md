前言
       mavon这款开源的markdown编辑器真的很不错，看得出作者很用心。在我重构网站之前，我使用的是一款叫做pagedown的开源编辑器，在github上也能找到，用的也很方便，如果不熟悉前端架构的，可以直接使用它，在html直接引入就行，熟悉前端的童鞋们我推荐这个mavon，比较适合颜值党。

重点
       废话后面就直接戳划重点了，mavon这个控件是支持自定义图片上传接口的，通过调用后端的图片存储接口，存储图片，然后调用方法$img2Url，将后台返回的url替换到原来图片的序号处。实现过程中我发现删除图片记录的时候没有把textarea中的图片记录删掉呢？因为本人是不搞明白不甘心的作死程序员，然后开始找问题，是不是我程序里出了问题，各个地方都打了日志，发现不是我自身代码的问题。

然后打开源码，在源码出打印一下日志信息

我们自定义的图片上传接口是由左侧工具栏的方法调用的，如下，这个num是从0开始的，而img_file: [[0, null]],初始定义是这样的，数组的第一个元素已经被占了，而且是个空的元素。

$imgFileAdd($file) {

this.img_file.push([$file, this.num]) 

this.$emit('imgAdd', this.num, $file)

this.num = this.num + 1

this.s_img_dropdown_open = false

}

根据上面的描述可以知道我们自定义的方法的第一个入参就是num，而num比img_file中的元素存储的index要小1，所以导致了后面调用$img2Url（实际上是调用$changeUrl(index, url) ）的空指针或是别的异常，实际上只需要给源码文件中的md-toolbar-left.vue中如下方法的index加上1即可。

$changeUrl(index, url) {

index = index + 1

this.img_file[index][1] = url

}



原创文章转载请标明出处

更多文章请查看

[http://www.canfeng.xyz](http://www.canfeng.xyz)