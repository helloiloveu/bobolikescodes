var express = require('express')
var fs = require('fs')
var bodypaser = require('body-parser')
var jwt=require('jwt-simple')
var multer = require('multer')
var path = require('path')
var dbo = require('./serverdbo')
var api = require('./const').API_URL
var port=require('./const').port
const uploadDir='../upload/'


var app = express()
app.use(bodypaser.urlencoded({
  extended: false
}))
app.use(bodypaser.json())



var upload = multer({
  dest: uploadDir
}) 


var accessLog=(api,comingIp)=>{
  console.log(api + ' has been accessed by %s at %s',comingIp,Date.now())
}

var verifyToken=(token)=>{
  try{
    var decoded=jwt.decode(token,'secrettt')    
  }
  catch (e){
    console.log('token wrong!')
    return false
  }
  return true
}
//task.add
app.post(api.task.add, (req, res) => {
  accessLog(api.task.add, req.ip)
  var token=req.get('token')
  if(!verifyToken(token))
    return res.sendStatus(401)
  var newTask = req.body.newTask
  if (newTask == null)
    return res.sendStatus(415)
  //todo: verify validity of newtask
  //todo: verify if the plugin designated exists in local disk, if not, return the message 'missing plugin' to the server 
  dbo.task.add(newTask, (err) => {
    err ? res.sendStatus(500) : res.sendStatus(200)
  })
})

//task.delete
app.post(api.task.del, (req, res) => {
  accessLog(api.task.del, req.ip)
  var token=req.get('token')
  if(!verifyToken(token))
    return res.sendStatus(401)
  var id = req.body.taskId
  if (id == null) 
    return res.sendStatus(415)
  dbo.task.del(id, (err) => {
    err ? res.sendStatus(500) : res.sendStatus(200)
  })
})


//plugin.add
app.post(api.plugin.add, upload.single('file'), (req, res) => {
  accessLog(api.plugin.add, req.ip)
  var token=req.get('token')
  if(!verifyToken(token))
    return res.sendStatus(401)
  var file = req.file
  try{
    fs.renameSync(uploadDir + file.filename, '../upload/' + file.originalname)
  }
  catch(e){
    res.sendStatus(500)
  }
  res.sendStatus(200)
})

//plugin.del
app.post(api.plugin.del, (req, res) => {
  accessLog(api.plugin.del,req.ip)
  var token=req.get('token')
  if(!verifyToken(token))
    return res.sendStatus(401)
  var pluginName = req.body.pluginName
  if (pluginName == null) 
    return res.sendStatus(415)  
  fs.unlink(uploadDir+'/'+pluginName, (err)=>{
    err ? res.sendStatus(500) : res.sendStatus(200)
  })
})
//plugin.get
app.post(api.plugin.get, (req, res) => {
  accessLog(api.plugin.get,req.ip)
  var token=req.get('token')
  if(!verifyToken(token))
    return res.sendStatus(401)
  let plugins
  try{
    plugins=fs.readdirSync(uploadDir)
  }
  catch(e){
    return res.sendStatus(500)
  }
  res.json(plugins)
})

//auth
app.post(api.auth,(req,res)=>{
  accessLog(api.auth , req.ip)
  var user=req.body.user
  var pw=req.body.password
  if(user==null||pw==null)
    return res.sendStatus(415)
  if (user=='xia'&&pw=='123'){
    var token = jwt.encode({user:'xia'}, 'secrettt');
    res.send(token)
  }
  else
    return res.sendStatus(401)
})

//start server at localhost on the designated port
var server = app.listen(port, function () {
  // var host = server.address().address
  // var port = server.address().port
  dbo.connect("mongodb://localhost:27017", 'cent', (err) => {
    err ? console.log('db connection fail!') : console.log('server starts!')
  })

})
