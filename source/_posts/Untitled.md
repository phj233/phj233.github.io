```bash
//lagrange
info: Lagrange.OneBot.Core.Network.Service.HttpService[3]
      Receive(Conn: 9d45aa0d-f9bd-4aca-8f71-579aba294b0f: {"action":"send_msg","params":{"message_type":"group","group_id":"261982336","message":[{"type":"node","data":{"user_id":"1920460866","nickname":"Any","content":[{"type":"image","data":{"file":"https://img3.saucenao.com/vpc/frame.php?expires=1722366000&auth=KIB-oGtMU7ZCMmcZ8uNf370kdF0&rsz=0&enc=aGVudGFpL0Jha3VueXV1IE1haWQgR2FyaS8yIC0gVm9sdW1lIDIgLSAjNjE3MDQwIChyYXcpKGNlbikubXA0LjI2MTgyLnppcA&sub=18748-75-781906.jpg"}},{"type":"text","data":{"text":"相似度: 41.16"}},{"type":"text","data":{"text":"\n标题: null"}},{"type":"text","data":{"text":"\n链接: [\"https://anidb.net/anime/6512\"] "}}]}}],"auto_escape":true}})

//napcat
收到http请求 /send_msg {"message_type":"group","group_id":"261982336","message":[{"type":"node","data":{"user_id":"1920460866","nickname":"Any","content":[{"type":"image","data":{"file":"https://img1.saucenao.com/res/pixiv/3947/manga/39476055_p24.jpg?auth=VYCkGvuwWZjGD_HcRt2fIg&exp=1722366000"}},{"type":"text","data":{"text":"相似度: 28.10"}},{"type":"text","data":{"text":"\n标题: 【腐向】ピアクリカレンダー（10月）"}},{"type":"text","data":{"text":"\n链接: [\"https://www.pixiv.net/member_illust.php?mode=medium&illust_id=39476055\"] "}}]}}],"auto_escape":true} 
```

确实是OB服务端问题，换了napcat就能正常发送了

