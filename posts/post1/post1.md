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



