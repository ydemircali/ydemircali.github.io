---
title: Nodejs ve MongoDB ile RabbitMQ consumer işlemi.
tags: [HowTo, RabbitMQ, Nodejs, MongoDB]
style: fill
color: success
description: Nodejs ve MongoDB ile RabbitMQ consumer işlemi
---

Merhaba,
  
[Bir önceki yazıda ](https://ydemircali.github.io/blog/AccountingRabbitMQ) yazılım hayatında değişen koşullar doğrultusunda, oluşan ihtiyaçlar ve bu ihtiyaçları neden hissettmemiz gerektiğine dair kısaca bahsetmiştik.
Aynı zamanda 2010'lu yıllardan bu yana hayatımıza giren ve yazılım sektöründe oldukça sık rastladığımız Mikroservis mimariler hakkında temel gereksinimden bahsetmiştik. Bu kısmı biraz daha irdeleyebiliriz. 
Mikroservisten önce veya mikroservislere ihtiyaç duymayan uygulamaları monolith yapıda kurguluyorduk/kurguluyoruz.<br>

Bu konuyu biraz hikayeleştirmek istersek, örneğin bir klinik merkezinde hasta bilgilerini excelde tutmak yerine artık bir yazılım geliştirilmesine kara verildi. Bir dns/hosting hizmeti satın alındı, database olarak ilişkisel database(mysql gibi) ve yazılım platformu olarak da .Net platformunu seçtik. 
Ürün sahibinden aldığımız bilgilerle uygulamamızı doktor, hasta, randevu gibi basit modüller halinde tasarlayıp tek bir paket halinde teslim ettik.<br>
Bir hasta ofise geliyor veya danışmanı arayarak randevu almak istiyor. Randevu sayfasından hasta, doktor ve gün saat gibi bilgiler girilip provizyon oluşturuluyor. 
Muayene günü geldiğinde doktor önüne hasta bilgileri geliyor, muayeneden sonra hastanın muayene/tedavi bilgileri girilip işlem sonlandırılıyor.
Basit bir sistem ile çok fazla doktor ve hastası olmayan dolayısıyla çok transaction işlemi olmayan bu uygulama gayet yeterli oluyor.<br>
Ürün sahibi çeşitli istekler doğrultusunda, siz de fix/feature geliştirmeler yaparak daha önce belirlemiş olduğunuz(böyle ufak yerlerde genelde prod takvimler pek olmuyor, kasap misali prod geçişleri oluyor gerçi :) )
uygulamayı canlıya alma günü ve saatinde yeni bir versiyon ile proda geçişinizi yaptınız. Monolith olarak tek bir paket halinde oluşturduğunuz bu uygulama gayet sağlıklı bir şekilde bu döngüde devam etti. <br>

Gelelim değişen şartlara veya zamanla ortaya çıkan ihtiyaçlara. <br>
Klinik merkezinde hasta sayısı gün geçtikçe artıyor, doktor sayısı artıyor, uygulamanın sayfaları geç yanıt veriyor, database transaction işlemleri gecikmeye başlıyor. 
Siz de uygulamanın gücü azaldı herhalde deyip :) dikey veya yatay ölçeklemelerle server/cpu/ram vs arttırarak uygulamanın iyileştiğini gördünüz. <br>
Aradan zaman geçti, klinikte bulunan radyoloji sonuçlarını dijitale aktarmak istediniz. Bu monolith uygulamanıza ek bir modul olarak devreye aldınız. 
Herşey güzel giderken uygulamanın yavaşladığını gördünüz, güçlü bir monitoring yapısı kurgulamadığınızdan deneme yanılma ile bir şekilde yavaşlamaya sebep olan modülün radyoloji modülü olduğunu farkettiniz.
Demek ki radyoloji gelince uygulamanın gücü azaldı deyip :) bir daha ölçekleme yollarını aradınız. <br>
Peki diğer modüller gayet güzel sorunsuz çalışıyorken sırf radyoloji modülü yavaş diye tüm uygulamayı ölçeklendirmek sizce doğru bir adım mıydı ?<br>
Yeni bir ihtiyaç geldi e-reçete yazılması gerekiyor, hadi diyelim sadece java sdk sı bulunan bir sisteme mahkum kaldınız ve bu modülü java ile yazmanız şart oldu.
Sırf bu modul için tüm uygulamayı java diline dönüştürüp yazmak sizce de doğru bir adım olur muydu ?<br>
Günler geçiyor eklenen yeni modüller ile birlikte uygulamanız büyüyor. Ürün sahibi de siz de artık şu cümleleri kuruyorsunuz, tek bir modülün bir sayfasında küçücük bir değişklik yüzünden tüm uygulamayı proda almak için uğraşıyoruz.
Bu cümleleri sık duymaya sebep olan monolith yapı, her seferinde deployment sürecini sancılı bir hale getirmiyor mu ? <br>

Buraya kadar anlatılan hikaye, aslında Mikroservis mimariye geçiş için gerekli sebepler ve ihtiyaçlardan ibaretti.<br>
Gereksiz yere ölçeklemeler, bir dil veya platforma bağımlı halde bulunma, sancılı deployment süreçleri vs. bunun gibi belki birçok neden daha mikroservis mimarinin gerekliliği hakkında sıralanailir.<br>

Tabi uygulamanızı mikroservis mimariye geçirmek veya sıfırdan mikroservis mimarisi ile yazmanın da kendine göre gereksinimleri vardır. En son örnek verdiğimiz deployment sürecinden bahsedelim. 
Uygulamanızı birçok mikroservise böldüğünüzde her bir mikroservis için farklı deployment süreci işletilmesi gerekir. Bunun için de güçlü bir CI/CD süreçlerini yöneten bir DevOps ekibinizin olması kaçınılmaz hal alır.<br>
Aksi halde her deployment için harcadığınız operasyonel eforlar uzun vadede içinden çıkılmaz hal alabilir. Güçlü bir DevOps ekibinin her mikroservis için kurguladığı automated release pipeline lar sayesinde deployment süreçleriniz daha başarılı hale geliyor.<br>
CI/CD pipeline toolları için Azure Pipeline, Jenkins, CircleCI, TeamCity gibi toollar örnek verilebilir. Bakınız :[https://www.katalon.com/resources-center/blog/ci-cd-tools/](https://www.katalon.com/resources-center/blog/ci-cd-tools/)  <br>

Biz ne yapmıştık :). Bir önceki yazıda basit muhasebe işlemini, asenkron yapıda kurgulayıp RabbitMQ'ya publish etmiştik. Bunun için bir client uygulamamız ve diğer clientların da olabileceği düşüncesiyle oluşturulan bir api tasarlamıştık.<br>
Daha önce bahsettiğimiz gibi kuyruk yapılarında bir de consumer yapısı vardır. Yani daha önce enqueue edilen, kuyruğa eklenen iletileri, o kuyruğa subscribe olup tüketen consumerların processi hakkında ufak bir örnek verelim.<br>
Muhasebe modülünüzü .Net platformunda ve MSSQL gibi bir ilişkisel veritabanı ile tasarlamıştık. Muhasebe işleminde kesilen fişlerin senkron olarak çalışması zorunlu olmadığından, var olan mikroservis yapımıza RabbitMQ ile asenkron bir yapı kazandırmıştık.
Bunu kurgularken de muhasebe fişlerinin izole bir yapıda olabileceğinin farkına vardık ve de MongoDB gibi bir NoSQL veritabanı kullandık. Kuyrukta bekleyen fişleri consume edip MongoDB ye yazacak olan uygulmayı da Nodejs ile tasarladık. Ee mikroservis sonuçta dilden bağımsızım :). 
Yeter ki konuşlandırdığımız yapılar doğru bir kararda olsun.
Eğer senkron çalışan ve immediate consistency dediğimiz anlık olarak bir tutarlığa ihtiyacınız varsa, bunun için kuyruk yapısı ve NoSQL veritabanı sizin için doğru bir seçim olmayabilir. <br>
Strong consistency, eventual consistency veya isolation kavramları gibi SQL-NoSQL konuları daha detaylı bir konu olduğundan burada bırakmak istiyorum. <br>
Tasarladığımız genel yapı :<br>
![](https://github.com/ydemircali/ydemircali.github.io/blob/main/_posts/images/accounting_microservice.PNG?raw=true) <br>

Peki Nodejs ? <br>
Microsoft'ta yer alan tanımı kullanmak istiyorum. (V8 Microsoft Edge'de de kullanılıyormuş, Nodejs tanımlarında V8'den bahsedilirken genelde Chrome diye örnek verilir sadece :) )<br>
[https://docs.microsoft.com/en-us/learn/modules/intro-to-nodejs/2-what](https://docs.microsoft.com/en-us/learn/modules/intro-to-nodejs/2-what)  Nodejs, JavaScript ile server side uygulamalar yazabileceğimiz, bir Javascript Runtime platformudur.
Yani JavaScript uygulamalarını veya kodlarını bir sunucu gibi bir tarayıcının dışındaki birçok yerde çalıştırmak için Node.js'yi kullanabiliriz.<br>
Node.js, Google Chrome, Opera ve Microsoft Edge dahil olmak üzere birçok tarayıcıya güç veren C/C++ ile yazılmıi V8 adlı bir JavaScript motoru üzerinde çalışır.<br>
V8 Engine'i Java'daki JVM'ye benzetmek yanlış olmaz herhalde. <br>

Bu kadar hikaye ve kısa bilgilerden sonra gelelim NodeJs ile RabbitMQ'dan nasıl dequeue/consume yaparız ona bakalım. <br>
RabbitMQ kendi sitesinde hem send hem de receive örneğini vermiş [https://www.rabbitmq.com/tutorials/tutorial-one-javascript.html](https://www.rabbitmq.com/tutorials/tutorial-one-javascript.html). 
Biz de buradan hareketle kendi consumer yapımızı kurmak istediğimizde aşağıdaki adımları kurgulayabiliriz.<br>

[Nodejs](https://nodejs.org/en/)'i kurduk ve Visual Studio Code'da (IDE farketmez :D ) bir klasör oluşturup index.js adında dosya oluşturalım.
Aşağıdaki komutlarla RabbitMQ işlemlerini sağlayacak amqp kütüphanesinin, web uygulama çatısı olan diğer nodejs modülü express kütüphanesinin npm paketlerini yüklüyoruz. .Net platformunda referans olarak eklediğimiz nuget paketleri gibi düşünebiliriz.

```bat
npm install amqplib
npm install express 
```	


```javascript
var amqp = require('amqplib/callback_api');

amqp.connect('amqp://localhost', function (errorAmqp, connection) {
    if (errorAmqp) {
        throw errorAmqp;
    }
    connection.createChannel(function (errorchannel, channel) {
        if (errorchannel) {
            throw errorchannel;
        }

        var queue = 'AccTransactions';
        channel.assertQueue(queue, {
            durable: false
        });
 
        console.log("Queue listening...", queue);
 
        channel.consume(queue, function (data) {
            item = JSON.parse(data.content.toString())
            console.log(item);
        },  { noAck: true });
    });
});
```

Yazdığımız index.js kodlarını node ile ayağa kaldırıyoruz. Burada Amqp ile localhostta bulunan AccTransaction kuyruğuna subscribe olduğumuzu söylüyoruz.

```bat
node index.js
```

![](https://github.com/ydemircali/ydemircali.github.io/blob/main/_posts/images/accounting_nodejs_rabbitmq.PNG?raw=true)
  
Test ettiğimde Nodejs 1000 kaydı sadece 2-3 sn gibi kısa bir sürede consume ettiğini farkettim. MongoDB'ye yazmayla birlikte toplamda 5-6 sn gibi bir süre aldı. Yani gereçekten Nodejs inanılmaz performanslı.<br>

Consume ile işlemi ile birlikte aldığınız dataları MongoDB'ye kaydetmeyi düşündük diyelim. MongoDB kurulumuna [şuradan](https://www.mongodb.com/try/download/community) bakabilirsiniz. <br>
Aynı projede aşağıdaki komut ile mongoDb işlemlerini yapacağımız mongoose npm pkaetini ekliyoruz.
```bat
npm install mongoose
```
MongoDB'de Accounting diye bir db oluşturuyoruz. Bu db altında Transactions adında doküman oluşturulur. Projemizde Models klasörü altında AccTransaction diye bir model oluşturalım. Bunu da bir schema ile bu dokümana bağlayalım.<br>
AccTransaction.js

```javascript
const mongoose = require('mongoose');

const transactionSchema = new mongoose.Schema({
  TransactionObjectId: {
    type: String
  },
  TransactionType: {
    type: String
  },
  Amount: {
    type: String
  },
})

module.exports = mongoose.model('AccTransaction',transactionSchema, 'Transactions')
```
Bu model daha önce oluşturduğumuz index.js 'de import edip kullanıyoruz. Kuyruktan aldığımız datayı modele convert edip Transactions dokümanına kaydediyoruz.<br>
index.js nihai ;

```javascript
var mongoose = require("mongoose");
var amqp = require('amqplib/callback_api');
const AccTransaction = require('./Models/AccTransaction')

mongoose.connect('mongodb://localhost/Accounting', 
{ useNewUrlParser: true, useUnifiedTopology: true, useCreateIndex: true });

amqp.connect('amqp://localhost', function (errorAmqp, connection) {
    if (errorAmqp) {
        throw errorAmqp;
    }
    connection.createChannel(function (errorchannel, channel) {
        if (errorchannel) {
            throw errorchannel;
        }

        var queue = 'AccTransactions';
        channel.assertQueue(queue, {
            durable: false
        });
 
        console.log("Queue listening...", queue);
 
        channel.consume(queue, function (data) {
            item = JSON.parse(data.content.toString())
            WriteMongo(item);
        },  { noAck: true });
    });
});

function WriteMongo(item) {

    var accTransactionItem = new AccTransaction(item);
    accTransactionItem.save();
    console.log("Added Queue TransacripnObjectId", accTransactionItem.TransactionObjectId);
  }
```
Herşey yolunda ise aşağıdaki gibi yine ayağa kaldırıp, gelecek olan dataları MongoDB'ye kaydetmek için kuyruğu dinliyoruz.
```bat
node index.js
```
![](https://github.com/ydemircali/ydemircali.github.io/blob/main/_posts/images/accounting_mongoDB.PNG?raw=true)

MongoDB toolu için [NoSQLBooster](https://nosqlbooster.com/) 'u kullanabilirsiniz.<br>

Mikroservisler, asenkron işlemler, bağımsız dil ve izole yapılar gibi konulara değindiğimiz yazının sonuna geldik. <br>
Bir başka yazıda buluşmak dileğiyle, sağlıcakla kalın.

Kaynak Kodlar :<br />
[Accounting.RabbitMQ.Consumer](https://github.com/ydemircali/Accounting.RabbitMQ.Consumer) <br />
  
 
