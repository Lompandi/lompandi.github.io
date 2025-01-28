# Minecraft 基岩版逆向工程 Part I
大約三年前，我開始接觸基岩版的逆向工程，並發現網路上相關的資源十分稀少。

因此，在接下來的幾篇文章中，我將以逆向Minecraft為主題，向大家分享一些遊戲插件開發技巧，例如DLL注入等方法。
# Summary
Minecraft 基岩板是 Minecraft 在被微軟併購後的產物，使用 C++ 20 作為開發語言，
Microsoft UWP 為 Windows 版本的開發基礎。

# Bare foot into the desert
在開始之前，這些是你需要準備的工具，他同時也是撰寫任何遊戲插件所需要的工具。

|  工具               | 用途                                                |
| ------------------- | --------------------------------------------------- |
| IDA                 | 靜態分析                                            | 
| Cheat Engine        | 動態分析，適合觀察資料於記憶體中的變化              | 
| Windbg              | 動態分析，配合偵錯檔案觀察程式執行邏輯              |


這是接下來會提到的專有名詞:
|                     |                                         意義                                            |
| ------------------- | ----------------------------------------------------------------------------------------|
| BDS                 | Bedrock dedicated server，為基岩板的官方伺服器軟體                                      |
| Sig                 | Signature，由位元組組成。Signature scanning 為插件用來尋找記憶體位址的一種方法            | 
| Client              | 在這指 ```Minecraft.Windows.exe```，為基岩板的遊戲執行檔                                |
| MCBE                | 全名 Minecraft Bedrock Edition，Minecraft 基岩板                                        |

# Reverse Engineering
首先先來說如何開始逆向工程，Bds直到 1.21.20 前都有提供偵錯檔案，而偵錯檔裏面包含了很多重要的資訊，

先給大家看一下有無偵錯檔的差異:

## 未使用偵錯檔
![使用偵錯檔](https://lompandi.github.io/posts/imgs/symbolized.png)
## 使用偵錯檔
![未使用偵錯檔](https://lompandi.github.io/posts/imgs/non-symbolized.png)

可以看到，有使用偵錯檔的哪張照片裡函數的名稱和類別名稱都有顯示，而這些都來自於 debug symbol，

也就是俗稱的 Symbol。

那麼，如果是要逆向 Client 的話，為甚麼要用到BDS呢 ? 那就和 MCBE 的 Codebase 有關了，雖然客戶端和伺服器端的主要邏輯不一樣，
但因為 MCBE 的 Codebase 太大了，以至於主要接收邏輯外還有很多代碼重疊，我們因此可以從這裡下手，
利用BDS的偵錯檔來或取關於 Client 的資訊。

## c'est parti !

這裡我們要餵給 IDA 三個執行檔

|   執行檔                           | 版本       |
|------------------------------------|------------|
|Bedrock Dedicated Server for Windows| 1.21.2.02  |
|Minecraft.Windows.exe               | 目前       |
|China Minecraft.Windows.exe         | 1.16.201   |

首先，下載 1.21.2.02 版本的 BDS for Windows:

https://www.minecraft.net/bedrockdedicatedserver/bin-win/bedrock-server-1.21.2.02.zip

解壓縮後，使用IDA分析執行檔。因為 BDS 執行檔很大，所以這可能需要花一點時間(以以往經驗大約1.5個小時左右)

初步分析完成後，我們也來用 IDA 分析 Client 

(檔案路徑一般在 ```C:\Program Files\WindowsApps\Microsoft.MinecraftUWP_1.21.5101.0_x64__8wekyb3d8bbwe\Minecraft.Windows.exe```，開啟"顯示隱藏

項目"後可以看到WindowsApps資料夾，之後改一下權限就可以存取了。

網上有教學:
https://www.youtube.com/watch?v=hEt86kPa0As)

還有一個Binary我們要分析，那就是 1.16.201 版的 China Client (中國版"我的世界")，他有提供舊版 Client 的 debug symbol，

其中有一些是 BDS 沒有給的:

https://www.mediafire.com/file/c2rw8416mw9xvln/China_Windows_Client_1.16.201.7z/file 

三個檔案都分析完後，就可以來初步的逆向工程了。

## ClientInstance
現代遊戲一般都有一個玩家X)，而在C++中通常會以一個類別來表示(e.g. Player, Actor...)

Minecraft 中也有 Player 類別，而本地玩家的 Player 類別又是基於 LocalPlayer 類別所衍伸出來的，
ClientInstance 及包含了LocalPlayer類別。另外ClientInstance 還包含很多額外的資料，如遊戲類別。
因為網上找不到相關的圖，所以我就自己畫了。

![MinecraftMainClassStructure](https://lompandi.github.io/posts/imgs/MCClass.drawio.png)

其中 LocalPlayer 的指標可以由 ClientInstance 的一個 getter ```getLocalPlayer```得到，而 ```ClientInstance``` 本身是 ```IClientInstance``` 介面的衍伸類別，```getLocalPlayer``` 則是 ```IClientInstance``` 的虛擬函數，所以一般為了方便這裡會用 Virtual Function Call 來呼叫 ```getLocalPlayer```
### Virtual Table
很多人接觸遊戲外掛撰寫時可能常常聽到 Vtable ，即 Virtual Table。 至於它到底是甚麼，下面就來簡單解說一下。

C++ OOP 功能裡面包含一個功能，即虛擬函數。其允許衍伸類別自定義或覆寫基礎類別的函數。聽起來可能有些深奧，

下面舉個例子:

```c++
class Creature {
public:
    virtual int get_legs() const = 0;

    virtual void speak(const std::string& message) const {
        std::cout << "Creature cannot speak." << std::endl;
    }
};

class Dog : public Creature {
public:
    int get_legs() const override { return 4; }
    void bark() { impl_bark(); }
    
private:
    void impl_bark() const {
        std::cout << "Woof! Woof!" << std::endl;
    }
};

class Human : public Creature {
public:
    int get_legs() const override { return 2; }

    void speak(const std::string& message) const override {
        std::cout << "Human says: " << message << std::endl;
    }
};
```
其中，```Creature``` 是基礎類別，而 ```Dog``` 和 ```Human``` 則是 ```Creature``` 的衍伸類別。

先來看基礎類別 ```Creature``` :
```c++
class Creature {
public:
    virtual int get_legs() const = 0;
    
    virtual ~Creature() = default;
    
    virtual void speak(const std::string& message) const {
        std::cout << "Creature cannot speak." << std::endl;
    }
};
```
第一個函式 ```get_legs``` 是所謂的"純虛擬函式"，即，其衍伸型別**必須**自行定義其函數內容。

因此，可以看到 ```Dog``` 以及 ```Human``` 類別都自行定義了 ```get_legs()```:
```
class Dog : public Creature {
public:
    int get_legs() const override { return 4; }
}

class Human : public Creature {
public:
    int get_legs() const override { return 2; }
}
```
可以看到，兩者的 ```get_legs()``` 實作並不一樣，這也是虛擬函數的用處，其允許衍伸型別自行定義虛擬函數。

再來看 ```Creature``` 的第二個函數 ```speak()``` 其包含了預設實做 (default implementation):
```c++
virtual void speak(const std::string& message) const {
    std::cout << "Creature cannot speak." << std::endl;
}
```

若衍伸型別沒有**覆寫** ```speak()``` 函數，則它會在呼叫時印出 ```Creature cannot speak.```

如 ```Dog``` 類別。反之，則執行覆寫函數的內容，如呼叫 ```Human``` 的 ```speak()``` 會印出 ```Human says: <message>```

在介紹完虛擬函數的主要功能後，我們來看一下編譯器是如何實做虛擬函數的。

### Virtual Table (Internal)

當一個延伸類別實作基礎類別時，編譯器會為其建立 ```virtual table thunk```，其包含了所有衍伸類別的虛擬函數位址，

**即虛擬函數表 (virtual table)**。 以剛剛的例子來說，最後的 ```virtual table thunk``` 在記憶體佈局可能如下:

![vtable-layout](https://lompandi.github.io/posts/imgs/vtable.drawio.png)

(p.s. 排列方式可能因編譯器不同而有所差異)

假設也程式碼使用了 Dog 和 Human:
```c++
int main(){
    Dog dog;
    Human human;

    Creature* creatures[] = { &dog, &human };
    
    for (Creature* creature : creatures) {
        std::cout << "Creature has " << creature->get_legs() << " legs." << std::endl;
        creature->speak("Hello!"); 
    }

    dog.bark();
    human.speak("Hello, world!");

    return 0;
}
```
我們先來看 IDA 反組譯後的 ```Human::get_legs```:
```c++
.text:004014EC ; int Human::get_legs()
.text:004014EC                 public __ZNK5Human8get_legsEv
.text:004014EC __ZNK5Human8get_legsEv proc near        ; DATA XREF: .rdata:off_4061EC↓o
.text:004014EC
.text:004014EC var_4           = dword ptr -4
.text:004014EC this            = dword ptr  8
.text:004014EC
.text:004014EC ; __unwind {
.text:004014EC                 push    ebp
.text:004014ED                 mov     ebp, esp
.text:004014EF                 sub     esp, 4
.text:004014F2                 mov     [ebp+var_4], ecx
.text:004014F5                 mov     eax, 2
.text:004014FA                 leave
.text:004014FB                 retn
.text:004014FB ; } 
.text:004014FB __ZNK5Human8get_legsEv endp
```
這裡幫大家複習一下 Windows x64 的 Calling Convention:

在呼叫一個函數以前，函數的前四個參數會分別對應到四個不同的暫存器，剩下參數的會直接透過堆疊傳入

|參數順序  | 暫存器    |
|----------|-----------|
|1         | rcx       |
|2         | rdx       |
|3         | r8        |
|4         | r9        |
|>4        | 堆疊      |
|回傳值    | rax       |

這裡我們看到 ```get_legs``` 先是壓入基底暫存 (ebp)， 之後把 stack top (esp 代表的位址) 向下移4個 BYTE
以騰出空間給 var_4，之後賦予 var_4 ecx(即第一個參數) 的值，然後把2放入 eax，接著 return。

到這裡可能會有人問 :

"```get_legs``` 不是沒有參數嗎? 為啥編譯後莫名其妙多出一個參數來了?"

這裡說明一下，C++中類別的成員函數 (Member Function) 如果沒被宣告為靜態 (即使用```static```關鍵字) 的話，編譯後產生的第一個參數將會是```this```指標，且編譯後不論```this```是否在函數中被使用，編譯器都會將其傳入。

回到正題，我們可以在 IDA 中由衍伸型別的 virtual function 找到其 Virtual Table，方法如下 :

1. 找到類別中任意一個 virtual function (以```Human```中的```speak```函數為例):

2. 定位到函數的開始位址，然後在IDA中用 ```Ctrl + X``` 或 ```View``` -> ```Open Subviews``` -> ```Cross References``` 來做交叉引用(Cross Reference)的查詢。
3. 在列表中，其中一筆結果會包含 ```dx_offset...```(x 可為 d, q，分別代表32位元指標和64位元)， 選取並定位到其位址。

4. 向上尋找，直到你看到 ```off_...```，其即為 Vtable 的起始位址




經由 g++ (C++11) 編譯後 IDA 產生的 ```main``` 偽代碼如下 (編譯器產生的 STL 模版代碼和解構子省略):
```c++
  p_argc = (Dog *)&argc;
  __main();
  v24 = &off_4061D4;
  v23 = &off_4061EC;
  v21 = &v24;
  v22 = &v23;
  v36 = (int *)&v21;
  CreatureIter = (int *)&v21;
  CreatureEnd = (int *)&v23;
  while ( CreatureIter != CreatureEnd )
  {
    pCreature = *CreatureIter;
    legs = (**(int (__fastcall ***)(int))pCreature)(pCreature);
    CreatureHas = std::operator<<<std::char_traits<char>>(&std::cout, "Creature has ");
    v5 = std::ostream::operator<<(CreatureHas, legs);
    v6 = std::operator<<<std::char_traits<char>>(v5, " legs.");
    std::ostream::operator<<(v6, &std::endl<char,std::char_traits<char>>);
    v7 = *(void (__thiscall **)(int, int *, int))(*(_DWORD *)pCreature + 12);
    /* ... */
    v15 = (char *)&v31 + 3;
    std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::basic_string(&v25);
    v7(pCreature, &v25, v10);
    /* destructors... */
    ++CreatureIter;
  }
  Dog::bark(v12);
  /* ... */
  v39 = (int)&v33 + 3;
  p_argc = (Dog *)"Hello, world!";
  std::__cxx11::basic_string<char,std::char_traits<char>,std::allocator<char>>::basic_string(v32);
  Human::speak(v32);
  /* destructors... */
  return 0;
 ```
 先來看
 
 




