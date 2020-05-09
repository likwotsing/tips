# Web上传的几种方式

## 表单上传

传统的form表单上传，使用form表单的input[type="file"]，可以打开系统的文件选择对话框，从而达到选择文件并上传的目的。

```js
<form method="post" action="http://uploadUrl" enctype="multipart/form-data">
    <input name="file" type="file" accept="image/gif,image.jpg" />
    <input name="token" type="hidden" />
    <input type="submit" value="提交" /> 
</form>
```

method="post",采用post方式提交数据

enctype="multipart/form-data"，采用multipart格式上传文件，此时request头会显示Content-Type: multipart/form-data;boundary=—-Web...

type="file"，使用input的file控件上传，如果是多文件批量上传，可添加multiple属性

accept，定义了input应该接受的类型，逗号分隔

触发事件：可以是input[type="file"]的change事件；也可以由一个独立的按钮的click提交整个表单。同时可以用input[type="hidden"]带一些其他参数，如token验证等。

## ajax无刷新上传

ajax无刷新上传的方式，本质上与表单上传无异，只是把表单里的内容提出来采用ajax提交，并且由前端决定请求结果回传后的展示结果，不用像直接表单上传那样刷新和跳转页面。使用jquery插件：

```html
<form>
    <input id="file" name="file" type="file" />
    <input id="token" name="token" type="hidden" />
</form>
```

```js
$('#file').on('change', function() {
    var formData = new FormData()
    formData.append('file', $('#file')[0].files)
    formData.append('token', $('#token').val())
    $.ajax({
        url: 'http://uploadUrl',
        type: 'post',
        data: formData,
        processData: false,
        contentType: false,
        success: function(res) {
            // 根据返回结果回显到页面
        }
    })
})
```

上面的例子使用了file控件的change事件触发上传，也可以使用某个按钮来触发表单提交。提交数据时，用到了formData对象来发送二进制文件，formData构造函数提供的append()，除了直接添加二进制文件还可以附带一些其他的参数，作为XMLHttpRequest实例的参数提交给服务端。

使用jquery提供的ajax方法发送二进制文件，还需要附件两个参数：

processData: false // 不要对data进行序列化处理，默认为true

contentType: false // 不要设置Content-Type，因为文件数据是以multipart/form-data来编码

## flash上传

上传需求要求**显示上传进度、中断上传过程、大文件分片上传**等，传统的表单上传很难实现这些功能，于是产生了flash上传的方式。它采用flash作为一个中间代理曾，代替客户端跟服务端通信，此外，它也对客户端的文件选择方面拥有更多的控制权，比input[type="file"]要大得多。

jquery封装好的uploadify插件，该插件自带了上传用的flash文件，因为跟服务端回传的数据和展示跟客户端的交互，都是flash文件的接口跟插件来对接的。

```html
<div id="file_upload"></div>
```

html部分很简单，预留一个hook后，插件会在这个节点内部创建flash的object，并且还附带了上传进度、取消空间和多文件队列展示等界面。

```js
$(function() {
    $('#file_upload').uploadify({
        auto: true,
        method: 'post',
        width: 120,
        height: 30,
        swf: './uploadfiy.swf',
        uploader: 'http://uploadUrl',
        formData: {
            token: $('#token').val()
        },
        fileObjName: 'file',
        onUploadSuccess: function(file, data, response) {
            // some code
        }
    })
})
```

flash并不适合手机端，更好的解决方案是使用flash+html5来解决平台的兼容性问题

## 截图粘贴上传

如WebUploader，支持qq截图然后粘贴上传。

截图粘贴上传的核心思想是：**监听粘贴事件**，然后获取剪切板中的数据，如果是一张图片，则触发上传事件。

```js
$('textarea').on('paste', function(e) {
    e.stopPropagation()
    var self = this
    var clipboardData = e.originalEvent.clipboardData
    if (clickboardData.items.length <= 0) {
        return
    }
    var file = clicpboardData.items[0].getAsFile()
    if (!file) {
        return
    }
    var formData = new FormData()
    formData.append('file', file)
    formData.append('token', $('#token').val())
    $.ajax({
        url: 'http://uploadUrl',
        type: 'post',
        data: formData
    }).done(function(response) {
        // do something
    })
    e.preventDefault()
})
```

当进行粘贴（邮件paste/ctrl +v）操作时，触发剪贴板事件paste，从系统剪切板获取内容，而系统剪切板的数据在不同浏览器保存在不同的位置：

IE：windows.clipboardData

其他：e.originalEvent.clipboardData

## 拖拽上传

拖拽上传拥戴了HTML5的新API，Drag和Drop，File API。

上传监听拖拽的三个事件：dragEnter、dragOver和drop，分别对应拖拽至、拖拽时和释放三个操作的处理机制，当然也可以监听dragLeave事件。

HTML5的File API提供了一个FileList的接口，可以通过拖拽事件的e.dataTransfer.files来传递的文件信息，获取本地文件列表信息。

File API有有一个FileReader对象，用以读取文件内容，并且可以监控读取状态，它提供的方法有readAsBinaryString, readAsDataURL, readAsText, abort等。

```js
$('#textarea').on('dragenter', function(e) {
    e.preventDefault()
})
$('#textarea').on('dragover', function(e) {
    e.preventDefault()
})
$('#textarea').on('drop', function(e) {
    e.stopPropagation()
    e.preventDefault()
    var files = e.originalEvent.dataTransfer.files
    _.each(files, function(file) {
        if (!/^image*/.test(file.type)) {
            return
        }
        var fileReader = new FileReader()
        fileReader.onload = function() {
            //uploadFile(file)
        }
        fileReader.readAsDataURL(file)
    })
})
```

- 在drop事件触发后，通过e.dataTransfer.files获取拖拽文件列表，在jquery中是e.originalEvent.dataTransfer.files

- 拖拽上传仅支持图片，文件对象中file.type标识了文件类型

- 由于可能是多图拖拽，所以遍历图片上传，使用了underscore的each方法
- 使用readAsDataURL读取文件内容为二进制文件，还可以将其转换为base64方式上传，只是http协议里存在对非二进制数据的上传大小限制为2M

## 拍照上传

手机实现拍照上传的代码：

```html
<input type=file accept="image/*">
<input type=file accept="video/*">
```

ios有拍照、录像、选取本地图片功能，部分android只有选取本地图片功能

## 上传与安全

前端验证：文件类型、后缀、大小等

后台：

- 后台需要进行文件类型、大小、来源验证
- 定义一个.htaccess文件，只允许访问指定扩展名的文件
- 将上传后的文件生成一个随机的文件名，并且加上此前生成的文件扩展名
- 设置上传目录执行权限，避免不怀好意的人绕过图片扩展名进行恶意攻击，拒绝脚本执行的可能性

