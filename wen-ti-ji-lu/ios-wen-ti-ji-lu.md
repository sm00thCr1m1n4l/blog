# IOS问题记录

* safari 日期格式 2019-1-1会无法识别，需要改成2019-01-01
* safari https跨域请求是需要手动指定返回头中的值Access-Control-Allow-Headers，不可以是\*， 例如， 

  _Access-Control-Allow-Headers:access-control-request-headers,allow,content-encoding,token_

  需要手动指定与服务端交互的请求头

* 文字渐变无法显示

```markup
<template>
    <div id="qq">
      ssssssssssssss
      <div>
        asdasd
      </div>
    </div>
</template>
<style lang="scss">
    #qq {
      background-image: linear-gradient(to bottom, #FEFCD3,#AA8354);
      -webkit-background-clip: text;
      -webkit-text-fill-color: transparent;
      color: #fefcd3;
    }
</style>

```

如上的文字渐变样式，给\#qq加了文字渐变样式之后，再往里边加一个div会导致所有文本都无法显示，将\#qq内部的div去掉或者改为span之后正常显示。（仅IOS，手头的设备版本12.4.5，iphone6，其他版本ios也有发现，PC，android无此问题）

