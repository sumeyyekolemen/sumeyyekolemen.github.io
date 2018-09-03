# NULLBYTE1 (VULNHUB)

NULLBYTE1 adlı makineye linkten ulaşabilirsiniz.
https://www.vulnhub.com/entry/nullbyte-1,126/

Makineyi indirip sanal makinede açtıktan sonra ip adresini bulmak için yerel ağda tarama yapılır. Bunun için aşağıdaki komutlardan bir tanesini kullanabilirsiniz.

`arp-scan --localnet`
`netdiscover`

![ip](images/ip.png)

Hedef ip adresi =192.168.1.103

(NOT: İlerleyen görsellerde ip adresinde farklılıklar olabilir farklı networklerden giriş yaptığım için)

Öncelikle hedefe erişim sağlayabilmek için port taraması yapmamız lazım. nmap aracı ile aşağıdaki gibi portlar taranır.Nmap hedef sistemde açık olan portları ve portlar üzerinde çalışan servisler hakkında bilgi sahibi olmamız sağlayan ağ tarama aracıdır.
Nmap için detaylı bilgi --> https://nmap.org/book/osdetect-usage.html

`nmap -sV -p- 192.168.1.103 `

-sV : Tarana portlarda hangi servislerin çalıştığını daha detaylı listelemeyi sağlar.

-p- : Tüm portları taraması için gereken parametredir.

![nmap](images/nmp.png)

Görüldüğü gibi 80,111,777,37272 portları açıktır;

80-http : Web servisinin çalıştığı port.
111-RPC : Sunucu ve istemci arasındaki iletişimi sağlayan RPC portokolünün çalıştığı port.
777-ssh : Uzak bağlantı sağlayan SSH' ın çalıştığı port.
37272- RPC : RPC' nin kullandığı portlardan bir tanesi.

Burada öncelikte web sayfasını kontrol etmekte fayda var.

![web](images/web1.png)

Görünürde birşey yok kodlarını da kontrol edelim.Aynı zamanda web sayfası dizinlerini taramak için dirb aracını ve yine web sayfasını taramak için arkada nikto aracını çalıştıralım.

`dirb http://192.168.1.103`

![dirb](images/dirb.png)

dikkat çeken dizinler:

phpmyadmin
uploads

Sayfanın kaynak kodlarını da kontrol edildiğinde dikkat çekecek birşey bulunmadı.Fakat main.gif' in icerisinde birşey saklanıp saklanmadığını kontrol etmek gerekir.

![exiftool](images/exiftool.png)

Yorum kısmında ki karışık karaktrleri dizine eklediğimizde aşağıdaki sayfa ile karşılasıyoruz.

![key](images/invalidkey.png)

Anahtar girilmesi isteniyor fakat elimizde herhangi birsey yok bruto force yönetimini denersek.

![hidra](images/hydra.png)

Elde edilen key'i denediğimizde bizi aşağıdaki sayfaya yönlendiriyor.

![use](images/us.png)

Rasgele deneme sonucunda:

![deneme](images/deneme.png)

Burada url'i test etmek için parametrenin sonuna '"' çift tırnak işareti ekledim ve sql hatası verdi.

![tırnak](images/tırnak.png)

Bunun üzerine sql injection denemeleri yapmak için sqlmap aracını kullandım.sqlmap aracı hakkında daha detaylı bilgi için https://www.netsparker.com.tr/blog/web-guvenligi/Ileri-Seviye-Sqlmap-Kullanimi/  gayet açık bir şekilde yardımcı olabilir

sıra ile önce veri tabanı kontrolü ve elde edilen veriler ile kullanıcı adı ve parolaya eriştik.

`sqlmap -u 10.5.40.134/kzMb5nVYJw/420search.php?usrtosearch=ffss `

![sql1](images/sq1.png)

`sqlmap -u 10.5.40.134/kzMb5nVYJw/420search.php?usrtosearch=ffss --dbs`

![sql2](images/sq2.png)

`sqlmap -u 10.5.40.134/kzMb5nVYJw/420search.php?usrtosearch=ffss -D seth -- tables`

![sql3](images/sq3.png)

`sqlmap -u 10.5.40.134/kzMb5nVYJw/420search.php?usrtosearch=ffss -D seth -T users --columns`

![sql4](images/sq4.png)

`sqlmap -u 10.5.40.134/kzMb5nVYJw/420search.php?usrtosearch=ffss -D seth -T users -C position,user,id,pass --dump`

![sql5](images/sq5.png)

ramses kullanıcı olduğuna ve paralo bilgisine veri tabanını dump ederek ulaştık. Parola bilgisi base64 ile encode edilmiş online base64 decoder kullanarak decode etdildi.

![decode](images/base64.png)

Buradan çıkan verinin hash olduğuna karar verip online hash cracker ile md5 ile hashlendiğini ve parolanın "omega" olduğunu öğreniyoruz.

![hash](images/hash.png)

Yazının ilk başında port kontrolü yapmıştık ve ssh portunu açık olduğunu biliyoruz. ramses kullanıcısı olarak uzak bağlantı yapmayı denediğimizde

![ssh](images/ssh.png)

ramses kullanıcısı olarak artık makinede işlemler yapabilir durumdayız.

ilk yapılacak olan kontrollerden biri olan geçmiş kontrol edildiğinde

![ls](images/ls.png)

dikkat çeken procwatch dosyasına yöneliyoruz. İzinlerini bulunduğu dizinleri inceliyoruz.

![proc](images/cdvar.png)

Çalıştırılabilir bir dosya olduğundan dolayı çalıştırıyoruz. procwatch dosyası aslında ps komutunu çalıştırdığını görüyoruz.

![pc](images/proc.png)

Aynı zamanda procwatch dosya SUID bitine sahip yani dosya aslında root izinleri ile çalışıyor bu yetki yükseltmek için kullanılabilecek bir yol.
Proc dosyası üzerinde biraz daha duruldugunda yaptığı şey aslında ps komutunu çalıştırmak ve bunu sistemden çekiyor. Linux sistemlerde bir komut çalıştırılacağı zaman komutun çalıştırılabilir dosyasının bulunması için environment daki PATH değişkenine atanmış yolların içinde komutun çalıştırılabilir dosyası olup olmadığı kontrol edilir.

![ps](images/env.png)

Burada ps komutu sırası ile PATH'in içindeki dizinlerde aranır.Kendi oluşturduğumuz bir ps dosyasının yolunu bu PATH değerinin ilk başına eklediğimizde procwatch dosyası ilk bulduğu yani bizim oluşturduğumuz ps dosyasını root yetkileri ile çalıştıracaktır.

`nano ps.c`

![nano](images/nano.png)

burada ki kod ile root yekileri ile bash komutunu çalıştırmayı amaçlıyoruz.Dosyayı oluşturup derledikten sonra export komutu ile PATH değişkenin başına ps dosyasının olduğu dizini veriyoruz.

`gcc ps.c -o ps`
`export PATH=/var/www/backup:<path in önceki değeri`

![ps](images/gcc.png)

PATH'i yeniden kontrol edip procwatch dosyasını yeniden çalıştırıyoruz.

![yeniproc](images/newpath.png)

![root](images/whoami.png)

![proof](images/proof.png)
