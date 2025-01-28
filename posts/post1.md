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
| Sig                 | Signature，由位元組組成。Signature scanning 插件用來尋找記憶體位址的一種方法            | 
| Client              | 在這指 ```Minecraft.Windows.exe```，為基岩板的遊戲執行檔                                |
| MCBE                | 全名 Minecraft Bedrock Edition，Minecraft 基岩板                                        |

# Reverse Engineering
首先先來說如何開始逆向工程，Bds直到 1.21.20 前都有提供偵錯檔案，而偵錯檔裏面包含了很多重要的資訊，

先給大家看一下有無偵錯檔的差異:

![使用偵錯檔](https://github.com/Lompandi/lompandi.github.io/blob/main/posts/imgs/symbolized.jpg)
![未使用偵錯檔](https://github.com/Lompandi/lompandi.github.io/blob/main/posts/imgs/non-symbolized.jpg)

