硌，当我们提交到AppStore 上的app 崩溃了，我想此时我们的内心也是崩溃的。此时，我们应该做的就是赶紧从iTunes store上下载crash log。可是crash log下载下来了，怎么知道crash log 和我们的二进制是对应的呢。               
每一个二进制都一个UUID，crash log 也包含一个UUID。那这两个UUID 有什么联系呢          
>注意：即使是相同的代码，相同的编译设置,重新编译也会产生不同的UUID。          



废话不多说，先来看看crash文件       
````
$ grep --after-context=2 "Binary Images:" ~/Desktop/crashlog/QQMusic.crash      
Binary Images:       
0x10004c000 - 0x102c57fff QQMusic arm64  <813351adc3743903b832ded6e23e7d62> /var/mobile/Containers/Bundle/Application/D4303251-B98D-4AAD-AA58-34E6478A0011/QQMusic.app/QQMusic        
0x103804000 - 0x103807fff MobileSubstrate.dylib arm64  <3134cfb2f722310ea2c742ae4dc131ab> /Library/MobileSubstrate/MobileSubstrate.dylib      
````



此时的 <b>813351adc3743903b832ded6e23e7d62</b> 就是这份crash log 对应的二级制的UUID了。再来看看二进制的UUID。       
```
$ xcrun dwarfdump --uuid qqmusic/Payload/QQMusic.app/QQMusic
UUID: 10D3D53D-CA6D-30D0-99FA-80468D423359 (armv7) qqmusic/Payload/QQMusic.app/QQMusic
UUID: 813351AD-C374-3903-B832-DED6E23E7D62 (arm64) qqmusic/Payload/QQMusic.app/QQMusic
```
现在看看，知道这份crash log 就是跟这个二进制是匹配，这样分析起来就容易一些了。
