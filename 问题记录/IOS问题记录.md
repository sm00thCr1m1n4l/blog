- safari 日期格式 2019-1-1会无法识别，需要改成2019-01-01


- safari https跨域请求是需要手动指定返回头中的值Access-Control-Allow-Headers，不可以是*， 例如， 
  Access-Control-Allow-Headers:access-control-request-headers,allow,content-encoding,token
  需要手动指定与服务端交互的请求头

