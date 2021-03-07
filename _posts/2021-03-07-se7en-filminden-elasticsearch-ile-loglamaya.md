---
title: Se7en Filminden ElasticSearch ile loglamaya
tags: [HowTo, ElasticSearch, Log, ASP.NET Core MVC, ASP.NET Core 5]
style:
color: success
comments: true
description: .Net Core MVC, ElasticSearh, Docker kullanarak kitaplık senaryosu ile kullanıcı hareketleri nasıl izlenir ?
---

Herkese selamlar,

Birkaç hafta önce yağışlar kesilmiş, kuraklık iyice hissetirilmişti. Yine iklim uzmanları haber programlarına çıkmış, küresel ısınmadan tutun yer altı barajlarına, 
binalarda yağmur suyu depolamaya kadar birçok konuyu tekrar gündeme getiriyorlardı. Pandemiden dolayı haftasonu sokağa çıkma yasakları olduğu için bazen açıp film izliyorduk. 
Yine hangi filmi izleyelim diye araştırırken, her zaman karşıma çıkan ama bir türlü izlemediğim Se7en filmini izleyelim dedik. 
1995 yapımı başrollerinde Morgan Freeman, Brad Pitt, Kevin Spacey olan filmin konusu, bir çaylak ve bir emektar olmak üzere iki dedektif, 
yedi ölümcül günahı kullanan bir seri katili avlıyorlar. Kaynak : [IMDB](https://www.imdb.com/title/tt0114369/).<br> 
Ama yaaniii filmde bir yağmur yağıyor bir yağmur yağıyor içim gitti. 
Prodüksiyon işi mi değil mi araştırmadım ama filmin her sahnesinde nerdeyse vardı ve kuraklık zamanında güzel oldu yani :)

![](https://github.com/ydemircali/ydemircali.github.io/blob/main/_posts/images/se7en.PNG?raw=true)

Filmde dedektifler seri katili araştırırken, katilin her cinayette bir kelime bıraktığını görüyor ve birkaç kelimeden sonra emektar ve tecrübeli dedektif Morgan abimiz bunların yedi ölümcül günahlara işaret ettiğini anlıyor. 
Ardından kütüphaneye gidip bu ölümcül günahlardan bahseden kitapları tek tek çıkarıyor ve anlamaya çalışıyor. İşte en çok hoşuma giden sahne geliyor... :)
Morgan abi daha önce iş yaptığı FBI'dan adamıyla iletişime geçiyor. Sonra adam gelip bir paket bırakıp gidiyor. Genç dedektif Brad abi ne oluyor diye sorunca, Morgan abi işte o cümleyi kuruyor... :)
"Kütüphaneye gelen kişilerin hangi kitabı okumak için ödünç aldığı vs. tutuluyor ve bu bilgiler FBI'ın bilgisayarların da saklanıyor."
Aldıkları bu istihbarat ile yedi ölümcül günah ile ilişkilendirdiği o kitapları okuyan kişilerin peşine düşüyorlar vs. film öylece devam ediyor. Güzel film tavsiye edilir, sonunu güzel bağlamışlar.

Bu film hikayesinde anlatılan FBI sahnesini bir yazılımcı olarak düşününce, aklıma loglama mekanizması geldi. Günümüzde hareket-ksiyon logları çok kıymetli. Data Mining yani veri madenciliği konusunda çok yardımcı oluyor.
Mesela, bunu google çok yapıyor, ziyaret ettiğiniz sitelerin bilgisini alıyor daha sonra ilişkili reklamları önünüze sunuyor. Ya da kurumsal bir uygulamada ekran logları tutulup, bu loglar işlenerek ekranlar ve kullanıcılar hakkında yorumlar yapılıyor, kararlar alınıyor.
Kullanılmayan ekranlar veya en çok kullanılan ekranlar vs vs...

Loglama denince modern teknolojilere bakınca akla ilk gelen ELK stack oluyor. E- ElasticSearch, L- Logstash, K-Kibana. Burada ElasticSearch bir NoSQL veritabanı, arama ve analiz motoru olarak görev alıyor. Logstash, aynı anda birden fazla kaynaktan veri alan, dönüştüren ve daha sonra Elasticsearch gibi bir stash-database'e gönderen, bir 
server‑side data processing pipeline olarak görev alıyor. Kibana ise Elasticsearch'te tutulan verileri çizelge ve grafiklerle görselleştirerek kullanıcaya sunmada görev alıyor. Detaylı bilgi [ELK Stack](https://www.elastic.co/what-is/elk-stack).

Bu çalışmada ağırlıklı olarak ElasticSearch ve Kibana tarafını kullanıyor olacağız, Logstash gibi bir pipeline'a şu aşamada ihtiyacımız bulunmuyor. Ama mesela bir kuyruk yapısı olsa bu kuyruktan Logstash'e logları consume edebilir, asenkron performans açısından faydalanabilirsiniz.

Bizim senaryomuz şöyle olacak : Login işlemleri yapılabilen bir web uygulaması olacak. Burada kitaplar listelenebilecek,
detay görüntüleme ve kullanıcı kendi kütüphanesine ekleyebilmek için oturum açması gerekecek.
Çünkü kullanıcı bilgileri ile loglar tutulacak. Admin olan kullanıcı ise kullanıcıların hangi kitapların detayına baktığı ve 
hangisini okumak için kütüphanesine eklediğini izleyebilecek.
Web uygulaması bir Asp.NET Core MVC uygulaması olacak, Docker üzerinde de ayağa kaldırdığımız ElasticSearch ve Kibana olacak.

Öncelikle aşağıdaki komut ile default olarak Login-Register işlemini SqLite kullanarak yapan mvc template ile projemizi oluşturuyoruz.
Proje oluştuktan sonra dotnet run ile projemizi ayağa kaldırıyoruz ve register-login sayfalarını default olarak geldiğini görebiliriz.

```bat
dotnet new mvc -n “LibraryJacob” -au Individual 
dotnet run
```	

![](https://github.com/ydemircali/ydemircali.github.io/blob/main/_posts/images/user_register.PNG?raw=true)

![](https://github.com/ydemircali/ydemircali.github.io/blob/main/_posts/images/user_login.PNG?raw=true)

Projemizde kullanacağımız ElasticSearch ve Kibana'yı docker compose aracı ile her iki uygulamayı container yapıda ayağa kaldırıyoruz. Daha önce docker kurulu değilse [şuradan](https://www.docker.com/products/docker-desktop) indirip kurabilirsiniz.
İki uygulamayı da docker hub'tan çekip ayağa kaldıracak docker-compose.yml adında yaml dosyasını oluşturabiliriz.

```yaml
version: '2.2'

services:
  
  elasticsearch:
   container_name: elasticsearch
   image: docker.elastic.co/elasticsearch/elasticsearch:7.11.1
   ports:
    - 9200:9200
   volumes:
    - elasticsearch-data:/usr/share/elasticsearch/data
   environment:
    - xpack.monitoring.enabled=true
    - xpack.watcher.enabled=false
    - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    - discovery.type=single-node
   networks:
    - elastic

  kibana:
   container_name: kibana
   image: docker.elastic.co/kibana/kibana:7.11.1
   ports:
    - 5601:5601
   depends_on:
    - elasticsearch
   environment:
    - ELASTICSEARCH_URL=http://localhost:9200
   networks:
    - elastic
  
networks:
  elastic:
    driver: bridge

volumes:
  elasticsearch-data:
```

Daha sonra aşağıdaki komut ile compose daki uygulamaları container olarak ayağa kaldırıyoruz. Ayağa kalkması için docker kurulu ve çalışıyor olması gerekiyor. Ayağa kalkan containerları VS Code'tan izleyebiliriz.
Detaylı bilgi : https://code.visualstudio.com/docs/containers/overview 

```bat
docker-compose up
```
ElasticSearch : http://localhost:9200/  , Kibana : http://localhost:5601/

![](https://github.com/ydemircali/ydemircali.github.io/blob/main/_posts/images/docker_containers.PNG?raw=true)

Ortamlarımız hazır olduğuna göre geliştirmeye başlayabiliriz. Projemizde index (RDBMS-table) tanımı yapan, bu indexe kayıtlar atan, bu indexten arama yapan base bir ElasticSearchService oluşturuyoruz.

```csharp
public abstract class ElasticSearchService<T> : IElasticSearchService<T> where T : BaseEntity
{
    private readonly ElasticClient _client;
    public ElasticSearchService(ElasticClientProvider provider)
    {
        _client = provider.ElasticClient;
    }

    public void CheckIndexExist(T logModel, string indexName)
    {
        var response = _client.Indices.Exists(indexName);
        if (!response.Exists)
        {
            _client.Indices.Create(indexName,
                 index => index.Map<T>(
                      x => x
                     .AutoMap()
              ));
        }

    }

    public IEnumerable<T> All(string indexName)
    {
        return _client.Search<T>(search =>
            search.From(0).Size(1000).MatchAll().Index(indexName)).Documents;
    }


    public bool Delete(Guid id)
    {
        throw new NotImplementedException();
    }

    public T Get(Guid id, string indexName)
    {
        var getResponse = _client.Get<T>(id, g => g.Index(indexName));

        return getResponse.Source;
    }

    public IndexResponse Save(T entity, string indexName)
    {
        CheckIndexExist(entity, indexName);

        var result = _client.Index<T>(entity, idx => idx.Index(indexName));
        return result;
    }
}
```

Base servisi kitap indexini kullanan BookServisi kullanacak. Ayrıca kullanıcların kitap aksiyonlarını da tutacağımız farklı bir index için de UserBookService kullanacak.
Öncelikle BooksController altında elimdeki sampla dataya sahip json dosyadan book adında index olarak ElasticSearch'e kaydeden bir method yazıyoruz. 
Kitapların kaydedildiği bir admin panel yapmadığım için kısayoldan bu yöntemi kullanıyoruz.

```csharp
public IActionResult FillDataFromJsonToElastic()
{
    var books = new List<BookJsonModel>();

    using (StreamReader r = new StreamReader("books_new.json"))
    {
        string json = r.ReadToEnd();
        books = JsonConvert.DeserializeObject<List<BookJsonModel>>(json);
    }

    foreach (var item in books)
    {
        var bookModel = new Book(){
            Id = Guid.NewGuid(),
            Title = item.Title,
            Author = BookHelper.TitleNameSurname(item.Author),
            Genre = BookHelper.GenreUpperCase(item.Genre),
            SubGenre = BookHelper.GenreUpperCase(item.SubGenre),
            Publisher = item.Publisher,
            Page = item.Height
        };   

        var response = _bookService.Save(bookModel, _bookIndexName);
        if (!response.IsValid)
        {
            throw new Exception(response.OriginalException.Message);
        }   

    }

    var booksElastic = _bookService.All(_bookIndexName);
    return Json(booksElastic);
}

```

BooksController Index methodunda bu kitap listesini çekip listeliyoruz.

```csharp
public class BooksController : Controller
{
    private string _bookIndexName;
    private string _userBookIndexName;
    private readonly BookService _bookService;
    private readonly UserBookService _userBookService;

    public BooksController(BookService bookService, UserBookService userBookService, IOptions<ElasticConnectionSettings> elasticConnection)
    {
        _bookService = bookService;
        _userBookService = userBookService;
        _bookIndexName = elasticConnection.Value.ElasticBookIndex;
        _userBookIndexName = elasticConnection.Value.ElasticUserBookIndex;
    }

    public IActionResult Index()
    {
        var books = _bookService.All(_bookIndexName);
        return View(books);
    }
}

```

![](https://github.com/ydemircali/ydemircali.github.io/blob/main/_posts/images/book_list.PNG?raw=true)

Detail methodu ile kitap listesinde bir kitabın detaya gidilmesini sağlıyoruz. Ayrıca bu aksiyonu userbook indexine kaydetmek için \[Atuhorize\] attribute ile kullanıcı oturum açmış olması gerektiğini sağlıyoruz.
Kullanıcı detaya girdiğinde detay aksiyonunu userbook indexine gönderiyoruz.

```csharp
[Authorize]
public IActionResult Detail(Guid id)
{
    var book = _bookService.Get(id, _bookIndexName);
    var userBooks = _userBookService.SearchUserBooks(_userBookIndexName,User.Identity.Name).ToList();
    ViewData["AddedLibrary"] = userBooks.Any(s => s.Action == "AddMyLibrary" && s.BookTitle == book.Title) ? "1" : "0";

    if (book != null)
    {
        var userBookModel = new UserBookModel(){
            Id = Guid.NewGuid(),
            UserName = User.Identity.Name,
            BookTitle = book.Title,
            BookCategory = book.Genre,
            BookSubCategory = book.SubGenre,
            BookAuthor = book.Author,
            Action = "Detail",
            ActionDate = DateTime.Now
        };

        var response = _userBookService.Save(userBookModel, _userBookIndexName);
        if (!response.IsValid)
        {
            throw new Exception(response.OriginalException.Message);
        }

    }

    return View(book);
}
```

![](https://github.com/ydemircali/ydemircali.github.io/blob/main/_posts/images/book_detail.PNG?raw=true)

Kibana dev tools sekmesinden console'a ulaşıp aşağıdaki bir query yazdığınızda kaydın düştüğünü görebiliriz.

![](https://github.com/ydemircali/ydemircali.github.io/blob/main/_posts/images/kibana_console.PNG?raw=true)

Detaya gidilip Add My Library'e tıklandığında ajax çağrısı ile arkada AddMyLibrary methodunu kullanarak userbook indexine bu kez AddMyLibrary aksiyonunu ekliyoruz. Aynı şekilde kibana console'da bu aksiyonu da görebiliyoruz.

```csharp
[HttpPost]
[Route("addMyLibrary")]
public JsonResult AddMyLibrary(Guid bookId)
{
    AjaxResult result = new AjaxResult();

    var book = _bookService.Get(bookId, _bookIndexName);
    var userBooks = _userBookService.SearchUserBooks(_userBookIndexName,User.Identity.Name).ToList();

    if (book != null && !userBooks.Any(a => a.Action == "AddMyLibrary" && a.BookTitle == book.Title))
    {
        var userBookModel = new UserBookModel(){
            Id = Guid.NewGuid(),
            UserName = User.Identity.Name,
            BookTitle = book.Title,
            BookCategory = book.Genre,
            BookSubCategory = book.SubGenre,
            BookAuthor = book.Author,
            Action = "AddMyLibrary",
            ActionDate = DateTime.Now
        };

        var response = _userBookService.Save(userBookModel, _userBookIndexName);
        if (!response.IsValid)
        {
            result.IsSuccess = false;
            result.Message = response.OriginalException.Message;
        }
        else
        {
            result.IsSuccess = true;
            result.Message = "Added Your Library";
        }
    }

    return Json(result);
}
```

![](https://github.com/ydemircali/ydemircali.github.io/blob/main/_posts/images/kibana_console_add.PNG?raw=true)


Son olarak da Admin tarafında basit bir listeleme yapıyoruz. Bu listelemede userbook'taki tüm aksiyonları listeleyip datatable halinde gösteriyoruz. Tabi ki burada çeşitli filtreler kullanılabilir.
Ben sadece index adına göre tüm kayıtları çektim, kullanıcı adına göre kitap ismine göre veya tarihe göre sorgu yazılabilir. Örneğin username e göre nasıl çekebiliriz.

```csharp
public IReadOnlyCollection<UserBookModel> SearchUserBooks(string indexName, string userName)
{
    var response = _client.Search<UserBookModel>(s => s
                    .Index(indexName)
                    .From(0)
                    .Size(1000)
                    .Sort(st => st.Descending(p => p.ActionDate))
                    .Query(q => q.Match(m => m.Field(f => f.UserName).Query(userName)))

    );
    return response.Documents;
}
```

Admin tarafı için tümünü listeliyoruz ve datatable ile şimdilik filtreleme işini yapabiliriz. Örneğin Manasa adlı eserde kim hangi aksiyonları almış öğrenebiliriz.

![](https://github.com/ydemircali/ydemircali.github.io/blob/main/_posts/images/admin_manasa.PNG?raw=true)

Ayrıca http://localhost:5601/app/management/kibana/indexPatterns  yolu ile Kibana üzerinden ElasticSearch te var olan indexler üzerinden index pattern oluşturup, yine Kibana üzerinden Discover sekmesi ile bu indexte olup bitenleri grafiklerle izleyebilirsiniz.

![](https://github.com/ydemircali/ydemircali.github.io/blob/main/_posts/images/index_pattern.PNG?raw=true)

![](https://github.com/ydemircali/ydemircali.github.io/blob/main/_posts/images/kibana_discover.PNG?raw=true)


Asp.Net Core MVC ile ElasticSearch kullanmayı ve kullanıcı aksiyonlarını loglayıp bu logları incelediğimiz bu yazının sonuna geldik. <br>
Bir başka yazıda buluşmak dileğiyle, sağlıcakla kalın.

Kaynak Kodlar :<br />
[LibraryJacob](https://github.com/ydemircali/LibraryJacob) <br />
  
 
