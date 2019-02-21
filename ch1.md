# Operating system organization

・OSに対する要請のkeyは、一度に、複数のactivityをサポートすることである。
・例えば、`fork()`system callを使うことで、プロセスは新しいプロセスをstartできる。
・OSはこのようなプロセスの間で、コンピュータのresourceをtime-shareしなければならない。
・たとえば、hardware processorsの数よりも多いプロセスが存在した場合でも、OSはすべてのプロセスをprogressにしなければならない。
・またOSは各プロセスを"isolate"な状態にしなければならない
・それは、もしとあるプロセスにバグがある場合、そのプロセスと依存関係にない他のプロセスには、そのバグが影響しないということ
・しかし、完全にisolateするのは強すぎます、なぜなら、process間のinteractも可能であるべきだからです。たとえばpipe
・こうしてOSは３つの要求を満たさなければなりません：
multiplexing, isolation, interaction.

・この章ではOSが、これら３つの要求を満たすために、どのようにorganizedされているか、その概要を述べる
・３つの要求を満たすために、多くの方法が存在することがわかっているが、このtextではmain streamのデザインである*monolithic kernel*に焦点を絞る。*monolithic kernel* は多くのUnix OSで用いられている。
・この章では、xv6が動き始めた時の、最初のプロセス生成を追跡しながら、xv6のデザインを紹介する
・そうすることでtextは、プロセスはどうやってinteractし、OSはどうやって３つの要求を満たしているか、このようなxv6が提供するすべての抽象化の実装を、ほんの少し、提供する
・ほとんどのxv6はspecial-casing the first processを避けている（？）、そして代わりに、xv6が標準的なoperationのために提供しないといけないコードを再利用している
・後の章でそれぞれの抽象化について詳細にのべる

・xv6はIntel 80386 or x86 processor上で動作している、そしてほとんどの low-level functionalityがx86-specificである
・この本では読者が（何かしらのarchitectureにおいて）少し機械語プログラミングに触れたことがあることを仮定する
・そしてこの本では、x86-specificなideaをそれが登場するたびに紹介する
・Appendix AはPC platformの概要を短時間でのべている

## Abstracting physical resources
・OSに出会ったときに、思う最初の疑問は、なぜOSはすべてをもっているのか？だろう
・それはsystem callをlibraryとして実装した（？）
・このプランでは、それぞれのapplicationは必要に応じて、app自身のlibraryを持つことができる
・appはhardware resourceを用いて直接interactでき、それらのresourceをそのappに対するbestな方法で使用できる。
・deviceに埋め込まれたOSやreal-time systemのOSにはこのように作られたものもある

・このようなlibraryアプローチの欠点は、一つ以上のappが動いているとき、appはwell-behavedである必要がある
・例えば、それぞれのappは、他のappを動かすために、定期的にprocessorを諦めないといけない
+++++
一つのappがprocessorを独占できない
+++++
・"cooperative" time-sharing schemeは、もしすべてのappがお互いを信頼でき、バグが全くなかったら、OKにしている。（ひとつのappがprocessorを独占できるということ？）
・より典型的には、各appはお互いを信頼せず、バグが有る、そのためcooperative schemeよりも、より強固なisolationが求められる

・強固なisolationを実現するために、hardware resourceへの直接なアクセスは禁止されている、そのかわり、serviceの中へとresourceは抽象化している(?)
・例えばappは、物理的なdisk sectorにreadしたりwriteしたりする代わりに、`open`, `read`, `write`, `close` sys callを用いてファイルシステムにinteractする。
・これはappにpathnameの便利さを与え、OSにdiskを管理させている

・同様にUnixはプロセス間で、ハードウェアプロセッサを透過的にスイッチし、必要に応じてレジスタの状態をsaveしたりrestoreしたりすることで、appはtime sharingについて意識しなくて良い
・このような透過性(transparency)は、たとえ、いくつかのappで無限ループがあったとしても、OSにプロセッサを共有することを可能にしている

・他の例として、Unixプロセスは、自身のメモリimageをbuild upするために、物理メモリに直接interactするのではなく、`exec`を使用する
・これはOSがメモリ内のどこにプロセスを配置するかを決定できるということである。もし、メモリがtightな場合、OSはプロセスのいくつかのdataをdisk上にstoreすることさえある。
・`exec`は、実行可能なprogram imageをstoreするために、userにfile systemの快適さを提供する

・Unixプロセス間の、多くのinteractはfdを通しておこわなれる
・fdは抽象的にfileの詳細を述べるだけでなく、simplifyなinteractの方法で定義される
・例えば、もし、ひとつのappがpipeline内で、failしたら、kernelはそのpipeline内の次のプロセスのためにend-of-fileを生成する
・おわかりの通り、sys call interfaceは、プログラマ間の快適さと、strong isolationの可能性を提供するために、注意深くデザインされている。
・Unix interfaceは、abstractなresourceというだけでなく、それがとても良いものであるということを示してさえいる

## User mode, kernel mode and system call

・強力なisolationはappとOS間の強力な隔たりが求められる。
・もし、appが間違いを起こすと、OSがfailする、または、appがfailしてしまう
・そのかわり、OSはfailしたappをclean upできるようになるべきであり、他のappを継続して動かせるべきである
・強力なisolationを達成するために、OSは、appがOSのデータ構造や命令にmodify(またはread)できない状況や、appが他プロセスのメモリにアクセスできない状況を、手配しないといけない

・プロセッサ(CPU)は、強力なisolationのためにhardware supportを提供する
・例えばx86 CPUは(他CPUと同様)、命令を実行する際に、２つのmodeがある：kernel modeとuser mode
・kernel modeでは、CPUは"privileged"命令の実行を許可されている
・例えば、disk(またはI/Oデバイス)へのreadやwriteはprivilegedな命令の一種である
・user modeのappがprivilegedな命令を実行しようとすると、CPUは命令を実行せず、kernel modeへとスイッチする。それはkernel mode内のソフトウェアに、そのappをclean upさせ、何も実行させないためである
・fig0-1はその仕組を図説している
・appはuser-modeの命令のみ実行でき（数を足すなど、）、それは*user space*内での実行と言われる。一方kernel内のソフトウェアはprivilegedな命令も実行でき、それは*kernel space*内での実行と言われる
・kernel space内で動くソフトウェアを*kernel*と呼ぶ

・disk上のfileをread or writeしたいappは、kernelがそうするようにtransitionしないといけない
・なぜなら、app自身はI/O命令を実行できないから
・CPUは、CPUをuser modeからkernel modeへ切り替える命令と、kernelによってい示されるentry pointへ入る命令を発行する
・(x86 CPUこのために`int`命令を使用している)
・一度CPUがkernel modeへ変わると、kernelはsys call達を有効化できるようになり、appがrequestされた操作を実行できるかどうか決定する、そして拒否するか、または実行する
・kernelが、transitionのためにentry pointをkernel modeへとsetしておくことが重要
・もし、appがkernelのentry pointをsetできたら、malicious(悪意のある)appがkernel内にenterできてしまう


## Kernel organization

・keyなdesign questionは、kernel modeで実行されるべきOSの部分とはなにか？である
・いち可能性としては、OSをkernel内に常駐させておくということである。そしてすべてのsys callの実装はkernel modeでrunすることになる
・この仕組を`monolithic kernel`という

・この仕組ではOSすべてが、fullなhardware privilegeをもってじっこうされる
・この仕組は便利である、なぜならOSデザイナーは、fullのhardware privilegeが必要でないOSの部分を決定する必要がないからである
・さらに、OSの異なるパーツを組み合わせるのが簡単になる
・例えば、OSは、file systemと仮想メモリsystemの両方で共有されうるbufferキャッシュもつであろうから

・`monolithic` OSの欠点はOSの異なるパーツ間のinterfaceが複雑になること
・従ってOSのdeveloperはミスしやすくなる
・monolithic kernel内では、ミスは致命的である。なぜならkernel modeでのエラーはkernel内のfailになりうるから
・もしkernelがfailすると、コンピュータはstopし、よってすべてのappはfailする
・そしてコンピュータはrebootしなければならない

・kernel内のミスのリスクをへらすために
OSデザイナーは、kernel modeで動くosのコード量を最小限にする。そしてuser modeでのOSのbulk(?)を実行する
・このkernelの仕組みを*microkernel*という