---
title: RabbitMQ and .Net Core
tags: [HowTo, Notes, RabbitMQ, .Net Core]
style: fill
color: success
description: Simple accounting receipt operations with RabbitMQ and .Net Core
---

RabbitMQ ile basit muhasebe fiş işlemleri

[Kaynak Kodlar](https://github.com/ydemircali/AccountingRabbitMQ)

1. Önce RabbitMQ kurulumu yapabilirsiniz. Gerekli adımlar ve konfigurasyonlar detaylı bir şekilde [burada](https://medium.com/@ademolguner/rabbitmq-nedir-nas%C4%B1l-kurulur-nas%C4%B1l-konfig%C3%BCre-edilir-ea596a7c1c08) açıklanmış.

2. [http://localhost:15672](http://localhost:15672) localinizde 15672 portundan rabbitmq dashboarda erişmeniz mümkün.

  ![](https://github.com/ydemircali/AccountingRabbitMQ/blob/master/queues.PNG?raw=true) 
    
3. RabbitMQ'yu görebildiyseniz projeyi git clone ile localinize klonlayıp derleyebilirsiniz. Derlediğinizde [https://localhost:5001/](https://localhost:5001/) ile proje ayağa kalkacaktır.
 Farklı modüllerden kuyruğa fiş göndereceğimiz için [https://localhost:5001/](https://localhost:5001/) bu url tarayııda yeni bir sekme ile tekrar açın, farklı modüllerin açıldığını görün.
 Daha sonra verileri girerek fiş kesin.
 
  ![](https://github.com/ydemircali/AccountingRabbitMQ/blob/master/krediler_modul.PNG?raw=true)
  ![](https://github.com/ydemircali/AccountingRabbitMQ/blob/master/atm_modul.PNG?raw=true)
 
4. Kesilen fişlerin RabbitMQ da kuyruğa eklendiğin görelim.

  ![](https://github.com/ydemircali/AccountingRabbitMQ/blob/master/fis_kuyrugu.PNG?raw=true)
  ![](https://github.com/ydemircali/AccountingRabbitMQ/blob/master/kuyruk_mesajlar.PNG?raw=true)
