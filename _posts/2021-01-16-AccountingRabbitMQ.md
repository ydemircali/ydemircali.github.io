---
title: RabbitMQ, .Net Core, Web Api, Microservice ile asenkron işlemler.
tags: [HowTo, Notes, RabbitMQ, .Net Core]
style: fill
color: success
description: RabbitMQ, .Net Core, Web Api, Microservice ile Basit Muhasebe işlemleri
---

Merhaba,
  
  Günlük hayatımızda veya iş hayatımızda, herhangi bir meslek erbabı eğer sorumluluklarının bilincindeyse veya ortaya güzel bir sonuç çıkarmak istiyorsa arayış içerisindedir.
Bu arayış aslında mükemmelliğe doğru bir arayıştır. Her zaman yaptığı işin mükemmel olmasını veya sorumluluğunun karşılığını tam verdiğini hissettirmek ister. Bu mükemmellik arayışında işini daha çok kolaylaştıracak veya işine kalite katacak materyalleri arayıp bulmaya ve bunları kullanmaya başlar. 
  
  Yazılım geliştirirken de bir problemin,bir hatanın çözümünü sağlayacak bir kod parçası veya bir teknoloji öğrendiğinizde oh be diye bir gevşeme bir yumuşaklık bir hoşluk hissedersiniz :) . Özellikle kullandığınız bir programlama dilinin her yeni versiyonunda bir problem veya bir ihtiyaç için çözümler üretilmesi yazılımcılar için mutluluk kaynağıdır.
  
  Örnek olarak Microsoft .Net 5 ile birlikte, Entitiy Framework Core'un da 5 versiyonunu yayınladı. Bu versiyonda çoğu özelliklerinden biri, tablolar arasında many-to-many ilişkisini siz belirtmeden kurabilmesi.
Önceden EF Core'da Code First ile ORM yapmaya çalışırken DbContext'i aşağıdaki gibi oluştururduk. İki class'a ek olarak üçüncü class ve DbContext'te OnModelCreating ile ilşkileri belirtiyorduk.

```csharp
public class Article
{
    public int Id { get; set; }
    public string Name { get; set; }
    public ICollection<Catogory> Categories { get; set; }
}

public class Category
{
    public int Id { get; set; }
    public string Name { get; set; }
    public ICollection<Article> Articles { get; set; }
}

public class ArticleCategory
{
    public int ArticleID { get; set; }
    public Article Article { get; set; }
    public int CategoryID { get; set; }
    public Category Category { get; set; }
}

public class NewsContext : DbContext
{
    public DbSet<Article> Article { get; set; }
    public DbSet<Category> Category { get; set; }
    public DbSet<ArticleCategory> ArticleCategory { get; set; }
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<ArticleCategory>().HasKey(c => new { c.CategoryID, c.PostID });
    }
}

```

Artık bunlara gerek kalmadan aşağıdaki gibi kısaca geçiyorsunuz.
```csharp
public class Article
{
    public int Id { get; set; }
    public string Name { get; set; }
    public ICollection<Catogory> Categories { get; set; }
}

public class Category
{
    public int Id { get; set; }
    public string Name { get; set; }
    public ICollection<Article> Articles { get; set; }
}

public class NewsContext : DbContext
{
    public DbSet<Article> Article { get; set; }
    public DbSet<Category> Category { get; set; }
}

```

  Bunun gibi yazılım hayatında daha birçok örnek verilebilir. Örnek olarak bir projenizde micro işlerden oluşan macro bir yapınız var. Bunların dependency durumlarını gördükçe düşündükçe ve bu bağımlılıklardan gelen hataları gördükçe öfkenizden microlara bölünüyorsunuz. Ama bir bakıyorsunuz ki bu macro yapıyı microlara dönüştürmek varken neden ben micro micro olayım ki :)
  
  Projenizin detaylı analizini çıkarıyorsunuz, birbirine aslında bağımlı olmaya ihtiyaç duymayan hatta asenkron bile çalışabilecek yapılar ortaya çıkarıyorsunuz. Çıkardığınız bu analiz sonucunda yapınızı micro servislere bölüp farklı programlama diliyle geliştirmekten tutun farklı deployment süreçlerine kadar biribirinden bağımsız hale getiriyorsunuz. Öyle ki bir de bu micro servisleri container yapıları ile örneğin Docker ile dockerize ederek sürekli yaşamını sürdüren, crash olsa bile diğer micro yapıları etkilemeyen ve kolaylıkla tekrar hayata döndürülebilen yapılar oluşturuyorsunuz. 
  
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
 
