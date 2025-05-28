# uppy

## 安装

```bash
$ npm install @uppy/core
```

从 CDN 加载。

```html
<!-- 1. Add CSS to `<head>` -->  
<link href="https://releases.transloadit.com/uppy/v4.13.3/uppy.min.css" rel="stylesheet">  
  
<!-- 2. Initialize -->  
<div id="uppy"></div>  
  
<script type="module">  
import { Uppy } from "https://releases.transloadit.com/uppy/v4.13.3/uppy.min.mjs"  
const uppy = new Uppy()  
</script>
```
## 用法

API: https://uppy.io/docs/uppy/

```javascript
import Uppy from '@uppy/core';  
  
const uppy = new Uppy();
```

使用插件

```javascript
import Uppy from '@uppy/core';  
import DragDrop from '@uppy/drag-drop';  
  
const uppy = new Uppy();  
uppy.use(DragDrop, { target: 'body' });
```

## addFile()

```javascript
uppy.addFile({  
name: 'my-file.jpg', // file name  
type: 'image/jpeg', // file type  
data: blob, // file blob  
meta: {  
// optional, store the directory path of a file so Uppy can tell identical files in different directories apart.  
relativePath: webkitFileSystemEntry.relativePath,  
},  
source: 'Local', // optional, determines the source of the file, for example, Instagram.  
isRemote: false, // optional, set to true if actual file is not in the browser, but on some remote server, for example,  
// when using companion in combination with Instagram.  
});
```

## upload()

```javascript
uppy.upload().then((result) => {  
  console.info('Successful uploads:', result.successful);  
  
  if (result.failed.length > 0) {  
    console.error('Errors:');  
    result.failed.forEach((file) => {  
      console.error(file.error);  
    });  
  }  
});
```