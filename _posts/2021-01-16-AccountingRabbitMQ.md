---
title: RabbitMQ, .Net Core, Web Api, Microservice ile asenkron işlemler.
tags: [HowTo, Notes, RabbitMQ, .Net Core]
style:
color: success
comments: true
description: RabbitMQ, .Net Core, Web Api, Microservice ile Basit Muhasebe işlemleri
---

Merhaba,
  
  Günlük hayatımızda veya iş hayatımızda, herhangi bir meslek erbabı eğer sorumluluklarının bilincindeyse veya ortaya güzel bir sonuç çıkarmak istiyorsa arayış içerisindedir.
Bu arayış aslında mükemmelliğe doğru bir arayıştır. Her zaman yaptığı işin mükemmel olmasını veya sorumluluğunun karşılığını tam verdiğini hissettirmek ister. Bu mükemmellik arayışında işini daha çok kolaylaştıracak veya işine kalite katacak materyalleri arayıp bulmaya ve bunları kullanmaya başlar. 
  
  Yazılım geliştirirken de bir problemin,bir hatanın çözümünü sağlayacak bir kod parçası veya bir teknoloji öğrendiğinizde oh be diye bir gevşeme bir yumuşaklık bir hoşluk hissedersiniz :) . Özellikle kullandığınız bir programlama dilinin her yeni versiyonunda bir problem veya bir ihtiyaç için çözümler üretilmesi yazılımcılar için mutluluk kaynağıdır.
  
  Finansal işlemleri çok fazla olan ve bunları muhasebeleştirmek zorunda olan bir yazılım ürününüzün olduğunu düşünün. Muhasebe işlemleri çoğunlukla asenkron bir yapıya ihtiyaç duyarlar. Örneğin ATM'ye gidip maaşınızı çektiğinizde banka tarafında bu kasa işlemlerinin o anda senkron olmasına gerek yoktur. Kaldı ki asenkron yapıldığında performans ve zaman açısında size birçok faydası vardır. Bankalar muhasebe işlemlerini genelde Queue(kuyruk) yapıları ile sürdürürler. Doğrudan bu işi yapmadım ama gözlemleyebildim :)
  
  RabbitMQ teknolojisini incelerken, .Net ile implemente edip kullanmak istediğimde aklıma bu muhasebe işlemleri geldi ve bu konu üzerinde basit bir örnek yapmak istedim.
  
  RabbitMQ ile ilgili olarak çok fazla kaynak ve detaylı yazılar bulabilirsiniz. Adem Olguner tarafından kaleme alınan [şu yazıyı](https://medium.com/@ademolguner/rabbitmq-nedir-nas%C4%B1l-kurulur-nas%C4%B1l-konfig%C3%BCre-edilir-ea596a7c1c08) inceleyip RabbitMQ hakkında bilgi sahibi olup, kurulumlarını yapabilirsiniz.
  
  .Net projemizin genel structure aşağıdaki gibi olacak. Micro servislere ayırıyoruz, RabbitMQ işlemlerini yapan micro servisimiz de bir web api olacak. 
  
  ![](https://github.com/ydemircali/ydemircali.github.io/blob/main/_posts/images/accounting_project_structure.PNG?raw=true)
  
  
  Birinci adım olarak Accounting.Client projesini ayağa kaldırıyoruz.
  
  ![](https://github.com/ydemircali/ydemircali.github.io/blob/main/_posts/images/accounting_ui.PNG?raw=true)
  
  UI'dan Api'ye aşağıdaki gibi istek atan Controller methodumuz.
  
```csharp
[HttpPost]
public async Task<IActionResult> PushAsync(FisModel fisModel)
{
    string json = JsonConvert.SerializeObject(fisModel);
    StringContent data = new StringContent(json, Encoding.UTF8, "application/json");

    var client = new HttpClient();

    var response = await client.PostAsync(base_url+"AddQueue", data);
    string result = await response.Content.ReadAsStringAsync();
            
    client.Dispose();
            
    return RedirectToAction("Index");
}
```

  Bir sonraki adımda Accounting.RabbitMQ.Api projemizi ayağa kaldırıyoruz. /swagger ile kontrol edebilirsiniz. AddQeue ile Post alan bir method göreceksiniz.
  
  ![](https://github.com/ydemircali/ydemircali.github.io/blob/main/_posts/images/accounting_api_swagger.PNG?raw=true)
  
  Burada requestleri AddQueue ile alıp RabbitMQService'e iletiyoruz. RabbitMQService aşağıdaki gibi iletileri alıp kuyruğa ekliyor.

```csharp
public class RabbitMQService
{
   public FisModel data;
   public RabbitMQService(FisModel _data)
   {
      this.data = _data;
   }
   public string Post()
   {
      var factory = new ConnectionFactory() { HostName = "localhost" };
      using (var connection = factory.CreateConnection())
      using (var channel = connection.CreateModel())
      {
         channel.QueueDeclare(queue: "Fiş Kuyruğu",
                              durable: false,
                              exclusive: false,
                              autoDelete: false,
                              arguments: null);

         var fisModel = JsonConvert.SerializeObject(data);
         var body = Encoding.UTF8.GetBytes(fisModel);

         channel.BasicPublish(exchange: "",
                              routingKey: "Fiş Kuyruğu",
                              basicProperties: null,
                              body: body);
         return "TransactionObjectId:" + data.TransactionObjectId+ " added queue.";
       }
    }
}
```

  Üçüncü adım olarak da daha önce kurmuş olduğunuz RabbitMQ'nun admin paneli ile UI projesini açarak denemeler yaptığınızda fişlerin kuyruğa eklendiğini göreceksiniz.
  
  ![](https://github.com/ydemircali/ydemircali.github.io/blob/main/_posts/images/accounting_demo.gif?raw=true)
  
  ![](https://github.com/ydemircali/ydemircali.github.io/blob/main/_posts/images/accounting_queue.PNG?raw=true)
  
  Bu çalışma ile asenkron bir ihtiyacımızı ,mesaj(process) kuyruğu sistemi olan RabbitMQ ile nasıl karşılayabiliriz sorusunun cevabının büyük kısmını incelemiş olduk. 
  RabbitMQ kurgusunun üç bileşeni bulunuyor. Biz mesajları(process) ileten Publisher yapısını ve bu iletileri alıp kuyruğa ekleyen yapıyı incelemiş olduk. 
  Üçüncü bileşen ise Consumer bileşeni yani kuyruktaki iletileri dinleyen veya bunları tüketen bileşen.
  
  Consumer yapısını bir sonraki yazıda EF Core kullanıp DB ye kaydeden bir yapı şeklinde devam etmek istiyorum.
  
  Görüşmek dileğiyle, sağlıcakla kalın.

  Kaynak Kodlar :<br />
  [Accounting.Client](https://github.com/ydemircali/Accounting.Client) <br />
  [Accounting.RabbitMQ.Api](https://github.com/ydemircali/Accounting.RabbitMQ.Api)
 
