# Windows Kernel Driver - 初步理解和提權探討


## I. Kernel Driver 是甚麼
「Kernel Driver」，即**核心模式驅動程式**，指的是與作業系統核心（Kernel）緊密結合並負責控制硬體設備或提供低層次操作的軟體元件。作業系統核心是作業系統中最基本且最關鍵的部分，它負責管理硬體資源、處理程序、記憶體管理等。Kernel Driver 主要**作為硬體和作業系統之間的橋樑**。



這些驅動程式通常是特定硬體的控制程序，例如顯示卡、網路卡、磁碟驅動器等，它們負責處理與硬體設備的通訊、資料傳輸及指令解釋等工作。Kernel Driver 是在核心層級運行，因此擁有比一般應用程式更高的權限。

以下是 Windows 作業系統中一些常見的 Kernel Drivers:
|名稱       |功能       |
|-----------|-----------|
|dxgkrnl.sys|管理 DirectX 應用程式與顯示卡 (GPU) 之間的互動，處理 3D 圖形渲染及硬體加速，特別是在 DirectX 11 和 12 中。
|wdmaud.sys|管理操作系統與音效硬體的通訊，處理音訊播放、輸入及錄音功能。|
|tcpip.sys | 實現 TCP/IP 協議棧(Stack)，管理網路通信，處理封包及路由等。|
|disk.sys|管理硬碟、SSD 等儲存裝置的低層次存取，協助資料的讀取與寫入。|


而 Microsoft 也有提供關於 Kernel Driver 的驅動程式套件 [WDK](https://learn.microsoft.com/zh-tw/windows-hardware/drivers/download-the-wdk)

## II.1 控制暫存器
如果平時有在碰逆向工程相關方面的話，對於 Rax, Rcx, Rbx, Rdx, Rsi, Rsp, Rdi, Rbp, R8 ~ R15

這些暫存器應該不陌生，那其實還有另一類暫存器，專門改變或控制CPU行為，稱為[控制暫存器](https://en.wikipedia.org/wiki/Control_register)，在 64 位元系統上存在 Cr0 ~ Cr4 和 Cr8 這六種控制暫存，其暫存內容儲存了可以控制CPU行為的資料，以64位元模式 (64-bit mode, long mode) 來說，結構如下:

![ref1](https://lompandi.github.io/posts/post3/imgs/062111_1434_x86x8664CPU1.png)

其中灰色區域為保留欄位。

### 重要欄位說明:
### Cr0: 

|位元|欄位名稱|意義|
|----|-|----|
|31  | PG |控制分頁機制 (Paging) 是否生效，若為 1，分頁機制生效，把線性位址轉換為物理位址；若為 0， 分頁機制無效，線性位址直接作為物理位址|
|0   | PE| 若為 0，則CPU處於真實模式 (Real Mode)，此時 使用分段機制，此時若將PG設置為 1 將引起一般保護錯誤 (GPF)|

### Cr2:
|位元|欄位名稱|意義|
|----|--------|----|
|63~0| -      |  存放[分頁錯誤](https://zh.wikipedia.org/zh-tw/%E9%A1%B5%E7%BC%BA%E5%A4%B1)的虛擬位址|

### Cr3 (PDBR):
|位元|欄位名稱|意義|
|----|--------|----|
|51 ~ 12|-    | 指定**4級分頁表**的**基底位址**|

### Cr4:
![cr4](https://lompandi.github.io/posts/post3/imgs/CR4.jpg)
|位元|欄位名稱|意義|
|----|--------|----|
|21  | SMAP   | 指定SMAP是否生效|
|20  | SMEP   | 指定SMEP是否生效|
|5   | PAE    | 指定[實體位址擴充](https://zh.wikipedia.org/zh-tw/%E7%89%A9%E7%90%86%E5%9C%B0%E5%9D%80%E6%89%A9%E5%B1%95)是否生效|

## II.2 記憶體布局和虛擬位址
在 Windows 系統中，核心模式和使用者模式的記憶體分區大致上如下:

|分區| x86 32 位元 Windows | x64 64 位元 Windows|
|------------|--------------|--------------------
|空指標指定值分區|0x00000000 ~ 0x0000FFFF，大小為 64KB | 0x00000000\`00000000 ~ 0x00000000\`0000FFFF，大小為 64KB
|**使用者模式分區**|0x00010000 ~ 0x7FFEFFFF，大小為約 2GB|0x00000000\`00010000 ~ 0x00007FFF\`FFFEFFFF，大小為約 128TB|
|禁入分區|0x7FFF0000 ~ 0x7FFFFFFF，大小為 64KB| 0x00007FFF\`FFFFF0000 ~ 0x00007FFF\`FFFFFFFFF， 大小為 64KB|
|**核心模式分區**| 0x80000000 ~ 0xFFFFFFFF，大小為約 2GB|0x00008000\`00000000 ~ 0xFFFFFFFF\`FFFFFFFF，大小為約 16EB

(表中 64 位元 Windows 記憶體分區以 Windows 10 為例)

1. #### 空指標指定值分區: 

空 (NULL) 指標指定值分區是處理虛擬位址空間中 0x0000000 ~ 0x0000FFFF
   
的閉區間，如果處理程序試圖讀去或寫入位於這一分區的記憶體位址，就會引發**存取違規** (Access Violation)。

2. #### 使用者模式 (User Mode) 分區

使用者模式分區是每個處理程序可以使用的虛擬位址空間。程式中用到的動態連結程式庫 (dll)

也會載入這一分區，但是**程式執行時還要透過記憶體管理單元 (MMU) 將虛擬位址映射為實體記憶體位址 (Physical Address)**，

處理程序可用的虛擬位址空間總量受物理儲存空間 (即實體記憶體) 含虛擬記憶體大小之和的限制。

雖然 64 位元程式理論上可以使用 128TB 的虛擬空間，但實際上作業系統目前並不支援這麼大的虛擬位址空間，一方面不需要，

另一方面管理和維護這麼大的位址空間需要較大的消耗。

3. #### 禁入分區

由 Windows 保留的一快虛擬記憶體位址區域。

4. #### 核心模式 (Kernel Mode) 分區

核心模式分區式作業系統程式的駐地，與執行緒排序、記憶體管理、檔案系統、網路支援以及**驅動裝置程式**相關的程式都

載入該分區。該分區中的所有程式和資料都被保護起來，如果一個應用程式 (位於使用者模式分區)

試圖讀去或寫入位於這一分區的記憶體位址，就會引發**存取違規**，導致系統強制結束該程式。

## - 虛擬位址(線性位址)

### 歷史
在**保護模式**引入之前（即 80286 處理器之前），**真實模式（Real Mode）** 是 x86 架構的唯一工作模式，而 真實模式並沒有虛擬位址和特權等級的概念，程式可以任意修改實體記憶體位址處的內容，包括系統程式。這帶給作業系統極大的安全問題。

Intel 在 1982 年推出的 80286 處理器中引入**保護模式（Protected Mode）**，首次引入是虛擬位址，並在後續的 80386 處理器中進一步完善。

### 原理
虛擬位址是通過硬體和作業系統將實體位址轉換而成的連續位址空間。在現代作業系統中，虛擬位址讓**每個程式都能擁有獨立的記憶體空間**，從而避免不同程式之間的記憶體衝突。這些虛擬位址最終會被轉換為實體位址。

使用四級分頁表的虛擬記憶體的組成如下:

![ref3](https://lompandi.github.io/posts/post3/imgs/Addressing.png)

```c++
union VIRTUAL_ADDRESS {
    struct {
        uint64_t Offset : 12;
        uint64_t PtIndex : 9;
        uint64_t PdIndex : 9;
        uint64_t PdPtIndex : 9;
        uint64_t Pml4Index : 9;
        uint64_t Reserved : 16;
    } u;
};
```

其中除保留欄位外，剩下的欄位都是用來轉換其到實體位址的，下面慢慢說。

## - 分頁表:

### 說明:

|名詞 |意思|
|-----|----|
|PML4 |**P**age **M**ap **L**evel **4**，四級分頁映射表|
|PDPT| **P**age **D**irectory **P**ointer **T**able，分頁目錄指標表|
|PD  | **P**age **D**irectory，分頁目錄表|
|PT  | **P**age **T**able，分頁表|

### 分頁表入口 (Page Table Entry, PTE)
尾端的 "E"(如PML4**E**、PT**E**) 代表 "Entry"，即分頁表的入口，每個入口大小為8個位元。

 PML4 表包含了 512 個 PML4E，所以這代表4級分頁表可以容納 512GB 的實體位址。

Windows 在 ```MMPTE_HARDWARE```  中定義了分頁表入口的結構:
```c++
union MMPTE_HARDWARE {
    struct {
        uint64_t Present : 1;               //P
        uint64_t Write : 1;                 //R/W
        uint64_t UserAccessible : 1;        //U/S
        uint64_t WriteThrough : 1;          //PWT
        uint64_t CacheDisable : 1;          //PCD
        uint64_t Accessed : 1;              //A
        uint64_t Dirty : 1;                 //D
        uint64_t LargePage : 1;             //PS
        uint64_t Available : 4;             //G
        uint64_t PageFrameNumber : 36;
        uint64_t ReservedForHardware : 4;   
        uint64_t ReservedForSoftware : 11;
        uint64_t NoExecute : 1;             //XD
    } u;
};
```

(P.S. **其實PML4、PDPT、PD、PT的入口都是這種結構**，只是名稱容易混淆)

分頁表入口結構包含硬體控制位元，可以控制分頁的屬性。

一些常用的硬體控制位元如下:
#### 1. 存在（P）:

決定線性位址在實體記憶體中的位置。如果沒有實體位址，則會發生分頁錯誤，且 MMU 將嘗試將位址映射到 RAM 中。
#### 2. 讀/寫（R/W）:

決定該分頁是否可以寫入。如果清除此位，則該分頁為唯讀。 CR0 暫存器的第 16 位元（寫入保護）決定此控制標誌是否套用於核心模式分頁，否則將套用於使用者模式分頁。
#### 3. 使用者/管理員（U/S）

表示對應分頁是否僅為核心模式或使用者模式。如該位元為1，則該分頁可由兩者輕鬆訪問，否則它將僅限於核心模式。如果為某個 PDE 設定了該位元，那麼該 PDE 下的所有分頁都將繼承該控制標誌，除非 PTE 覆寫該標誌。
#### 4. 分頁直寫 ( PWT ):

控制分頁的快取模式以及使用[直寫](https://en.wikipedia.org/wiki/Cache_\(computing\)#WRITE-THROUGH)或回寫快取。如果設定了此位，則使用直寫快取。

#### 5. 分頁快取停用 ( PCD ) : 

顧名思義， 用於啟用或停用分頁快取。

#### 6. [髒（D）位元](https://en.wikipedia.org/wiki/Dirty_bit):

指示該分頁已被寫入；通常髒分頁必須先重設後才能被另一個程序使用。

#### 7. **分頁大小 ( PS )**:

若為1，其所對應的分頁為**大分頁 (Large Page, 4MB)**，反之則為**超大分頁 (Huge Page, 1GB)**。

#### 8. 全域（G）:

用於決定當 CR3 清除或變更時 TLB 快取刷新 的位置。只有當 CR4 暫存器中的分頁全域啟用 (PGE) 位元為 1 時，此控制位元才適用。其允許多個位址空間共享相同的分頁。

#### 9. 執行停用（XD，又稱NX）:

僅在處理器以**長模式**運作時適用，但它決定了指令可以在指定分頁內的任何位置執行。僅當 EFER 暫存器中的 NXE 位元為 1 時時才支援此功能。

## II.3 虛擬位址到實體位址的轉換

![ref2](https://lompandi.github.io/posts/post3/imgs/x64-paging.png)

控制暫存器 CR3 負責指定4級分頁表(PLM4)。

* ### 步驟一
將 CR3 轉換為```MMPTE_HARDWARE```，將其 ```PageFrameNumber```  乘以**4096**，得到**PML4 的基底實體位址**。

再將要轉換的虛擬位址中的 PML4 Index 乘以 8，加上 PML4 的基底實體位址，得到 **PML4E 的實體位址**。

* ### 步驟二
從 PLM4E 的實體位址讀入 8 個位元組並轉換為```MMPTE_HARDWARE```，將其 ```PageFrameNumber``` 乘以 4096，得到 **PDPT 的基底實體位址**。

將要轉換的虛擬位址中的 PDPT Index 乘以 8，加上 PDPT 的基底實體位址，得到 **PDPTE 的實體位址**。

* ### 步驟三
從 PDPTE 的實體位址讀入 8 個位元組並轉換為```MMPTE_HARDWARE```，將其 ```PageFrameNumber``` 乘以 4096，得到 **PD 的基底實體位址**。

* ### 步驟四(1) (PDPTE 的 PS 旗標為 1)，完成計算
這時如果 PDPTE 的 PS 旗標（位元 7）為 1，無須分頁表。直接將 PD 的基底實體位址與虛擬位址中前 30 個 位元相加，**即可直接對應所求的實體記憶體位址**。

* ### 步驟四(2) (PDPTE 的 PS 旗標為 0)
如果 PDPTE 的 PS 旗標為 0 ，那麼就像前兩次所做的那樣，將虛擬位址中的 PD Index 乘以 8 個然後加上 **PD 基底實體位址**，得到 **PDE 的實體位址**。

* ### 步驟五
從 PDE 的實體位址讀入 8 個位元組並轉換為```MMPTE_HARDWARE```，將其 ```PageFrameNumber``` 乘以 4096 得到**PT 的基底實體位址**。

* ### 步驟六(1) (PDE 的 PS 旗標為 1)，完成計算
如果 PDE 的 PS 旗標為 1，直接將虛擬位址中前 28 個位元和 **PT 基底實體位址** 相加，**即可直接對應所求的實體記憶體位址**。

* ### 步驟六(2) (PDE 的 PS 旗標為 0)
反之，如果 PDE 的 PS 旗標為 0，則將虛擬位址中的 PT Index 乘以 8 個然後加上 **PT 基底實體位址**，得到  **PTE 的實體位址**。之後，

* ### 步驟七，完成計算
從 PTE 的實體位址讀入 8 個位元組並轉換為```MMPTE_HARDWARE```，將其 ```PageFrameNumber``` 乘以 4096 加上虛擬位址的 Offset，**即可得到所求的實體記憶體位址**。

有分支計算的部份可能有些眼花撩亂，幫大家整理一下:

#### 1. *if* PDPTE.PS is *1* -> *Physical Address* = *PD Base* + *Virtual Address*[0:29]
#### 2. *if* PDE.PS is *1* ->   *Physical Address* = *PT Vase* + *Virtual Address*[0:27]

幸運的是，我們不必手動計算每個分頁結構表，因為 kd 中的!pte指令可以直接幫我們計算。

```
kd> !pte fffff802`7c000000
                                           VA fffff8027c000000
PXE at FFFFBE5F2F97CF80    PPE at FFFFBE5F2F9F0048    PDE at FFFFBE5F3E009F00    PTE at FFFFBE7C013E0000
contains 0000000003F09063  contains 0000000003F0A063  contains 8A000000020001A1  contains 0000000000000000
pfn 3f09      ---DA--KWEV  pfn 3f0a      ---DA--KWEV  pfn 2000      -GL-A--KR-V  LARGE PAGE pfn 2000
```

其中每條虛線對應一個特定的控制位元。PXE 部分實際上是指該線性位址的 PML4 表，而 PPE 則表示分頁 PDP 表。(為甚麼要亂給人家改名字??😭)

那接下來我們就使用上述方式來手算 ```fffff802`7c000000``` 的 Physical Address。

手先，輸入 ```r cr3``` 取得cr3的值:
```
kd> r cr3
cr3=00000000001ad000
```
先將 Virtual Address 轉換，得到分頁表的索引值和偏移值:
```
Offset: 0
PtIndex: 0
PdIndex: 1e0
PdPtIndex: 9
Pml4Index: 1f0
```

取其 12~48 個位元，即 ```0x0000001ad```，乘以 4096，得到 ```0x00000000001ad000```。加上 ```Pml4Index x 8```，即 ```0x1F0 x 8 = 0xF80```，得 PML4E 的位址在 ```0x00000000`001adf80```。

接下來，使用```!dq 0x00000000`001adf80 L1``` 來取得實體位址 ```0x00000000`001adf80``` 的指標。(q 代表 quadword，在 64 位元架構上是 8 個 byte)

```
kd> !dq 0x00000000`001adf80 L1
#  1adf80 00000000`03f09063
```
得到 PML4E ```0x00000000`03f09063```。 和 !pte 的 PXE 對應一下，發現我們計算正確。

接下來，取```0x00000000`03f09063```的 12~48 個位元，乘以 4096，得 ```0x00000000`03f09000```。加上 0x48 (即 PdPtIndex x 8)，得到 PDPTE 的位址在 ```0x00000000`03f09048```:
```
kd> !dq 0x00000000`03f09048 L1
# 3f09048 00000000`03f0a063
```
得到 PDPTE ```00000000`03f0a063```。 和 !pte 的 PPE 對應一下，發現我們計算正確。

接下來，先將 PDPTE 轉換成```MMPTE_HARDWARE```:
```
Present: 1
Write: 1
UserAccessible: 0
WriteThrough: 0
CacheDisable: 0
Accessed: 1
Dirty: 1
LargePage: 0
Available: 0
PageFrameNumber: 3f0a
NoExecute: 0
```
發現其 LargePage 設為 0，故繼續計算，
取```00000000`03f0a063```的 12~48 個位元，乘以 4096，得 ```0x00000000`03f0a000```。加上 0xF00 (即 PdIndex x 8)，得到 PDE 的位址在 ```0x00000000`03f0af00```:

```
kd> !dq 0x00000000`03f0af00 L1
# 3f0af00 8a000000`020001a1
```
得到 PDE ```8a000000`020001a1```，將其轉換成```MMPTE_HARDWARE```:
```
Present: 1
Write: 0
UserAccessible: 0
WriteThrough: 0
CacheDisable: 0
Accessed: 1
Dirty: 0
LargePage: 1
Available: 1
PageFrameNumber: 2000
NoExecute: 1
```
發現其 LargePage 設為 1，故取```0x8a000000`020001a1```的 12~48 個位元，乘以 4096，得 ```0x02000000```。

並將其加上 Virtual Address 的 1 ~ 28 位元，得轉換後的實體位址 = 0x02000000 + 0 = **0x02000000**。

接著使用```dq```和```!dq```來比較 Virtual Address 和 Physical Address 的資訊，並檢查計算結果:
```
kd> dq fffff802`7c000000
fffff802`7c000000  00000003`00905a4d 0000ffff`00000004
fffff802`7c000010  00000000`000000b8 00000000`00000040
fffff802`7c000020  00000000`00000000 00000000`00000000
fffff802`7c000030  00000000`00000000 00000118`00000000
fffff802`7c000040  cd09b400`0eba1f0e 685421cd`4c01b821
fffff802`7c000050  72676f72`70207369 6f6e6e61`63206d61
fffff802`7c000060  6e757220`65622074 20534f44`206e6920
fffff802`7c000070  0a0d0d2e`65646f6d 00000000`00000024
kd> !dq 0x02000000
# 2000000 00000003`00905a4d 0000ffff`00000004
# 2000010 00000000`000000b8 00000000`00000040
# 2000020 00000000`00000000 00000000`00000000
# 2000030 00000000`00000000 00000118`00000000
# 2000040 cd09b400`0eba1f0e 685421cd`4c01b821
# 2000050 72676f72`70207369 6f6e6e61`63206d61
# 2000060 6e757220`65622074 20534f44`206e6920
# 2000070 0a0d0d2e`65646f6d 00000000`00000024
```

可以發現，兩者包含的資訊相同，故我們的轉換計算正確 <span style="font-size: 24px;">😆</span> 。

## III. 特權提升 (Priveledge Esclation) 
那之前說到了，Kernel Driver 有比一般應用程式更高的權限，那 Windows 作業系統是如何辨別每個程序的權限呢 ?

這就和**存取權杖** (Access Token) 有關了。

* ### Access Token

存取權杖 (Access Token) 簡單來說就是一個用來描述程序或執行緒的資料，當使用者登入時，系統會產生存取權杖，

並以此識別程序或執行緒的使用者**以及其權限**。如果要更詳細的資訊，可以參考由 Microsoft 提供的[參考文件](https://learn.microsoft.com/zh-tw/windows/win32/secauthz/access-tokens)。

實際上，Access Token 是有可能被篡改的，這意味著你可以將一個具有更高權限(如，系統權限)的 Access Token 嵌入到你的程序中，從而使你的程式獲得與該高權限程序相同的權限。這就是 Windows 中**特權提升**(或提權)的常用做法。

我們可以用 kd 來**模擬**提權的過程，首先，在虛擬機器中開啟一個 CMD，接下來在CMD中輸入 ```whoami```:
```
C:\Users\seantzou>whoami
desktop-xxxxxxx\seantzou
```

你會看到的使用者的名稱，這代表當前CMD是以使用者的權限執行的。

接下來在 kd 中輸入 ```!process 0 1 cmd.exe```:
```
kd> !process 0 1 cmd.exe
PROCESS ffffe60fced33080
    SessionId: 1  Cid: 14a0    Peb: 2d9e8d7000  ParentCid: 1140
    DirBase: 1f64fb000  ObjectTable: ffffbe8c99224d00  HandleCount:  73.
    Image: cmd.exe
    VadRoot ffffe60fd27d2dd0 Vads 36 Clone 0 Private 239. Modified 0. Locked 8.
    DeviceMap ffffbe8c493ba900
    Token                             ffffbe8ca37e5670
    ElapsedTime                       00:03:07.148
    UserTime                          00:00:00.000
    KernelTime                        00:00:00.000
    [...]
```

我們可以看到有一個 ```Token``` 欄位，那就是 cmd.exe 的 Access Token。接下來，我們需要找一個有系統權限的程序並拷貝他的 Token, 但是要找哪個程序呢? 其實，在 Windows 中，有一個system 程序，PID 恆為 4 ，他就有系統權限的 Token，我們來看一下:
```
kd> !process 4 1
Searching for Process with Cid == 4
PROCESS ffffe60fc927b040
    SessionId: none  Cid: 0004    Peb: 00000000  ParentCid: 0000
    DirBase: 001ad000  ObjectTable: ffffbe8c44c5dec0  HandleCount: 2819.
    Image: System
    VadRoot ffffe60fc92a5790 Vads 6 Clone 0 Private 22. Modified 155603. Locked 0.
    DeviceMap ffffbe8c44c39300
    Token                             ffffbe8c44c4c080
    ElapsedTime                       01:47:28.365
    UserTime                          00:00:00.000
    KernelTime                        00:00:14.515
    [...]
```
這樣，擁有系統權限的 Token 就到手了!那麼，我們如何把 cmd.exe 的 Token 竄改成 system 的呢?

我們之前輸入 ```!process <...>``` 時，會出現一個 ```PROCESS <...>```，它指向一個叫 EPROCESS 的結構。每個程序都有自己的 EPROCESS 結構，裡面儲存了 Access Token。

可以在 kd 中輸入 ```dt nt!_EPROCESS``` 來查看 ```EPROCESS``` 的整體結構。
```
kd> dt nt!_EPROCESS
   +0x000 Pcb              : _KPROCESS
   +0x438 ProcessLock      : _EX_PUSH_LOCK
   +0x440 UniqueProcessId  : PVOID
   +0x448 ActiveProcessLinks : _LIST_ENTRY
   [...]
   +0x4b8 Token            : _EX_FAST_REF
   [...]
```
其中，Access Token 被儲存在基底偏移 0x4b8 的位址，為一個 ```_EX_FAST_REF``` 結構。

以 system 程序為例，我們只需將 ```PROCESS <...>``` 的值加上 0x4b8，並使用 ```dt _EX_FAST_REF <位址>``` 來查看其資料:
```
kd> dt _EX_FAST_REF ffffe60fc927b040+4b8
nt!_EX_FAST_REF
   +0x000 Object           : 0xffffbe8c`44c4c08c Void
   +0x000 RefCnt           : 0y1100
   +0x000 Value            : 0xffffbe8c`44c4c08c
```
你會發現 ```Value```欄位的值跟之前看到的 Token 不太一樣，那是因為 ```_EX_FAST_REF``` 會將其參考數量(Reference Count) 處存於 Value 的前四位元，所以將其 mask 掉以後，就可以得到真正的 Token 值了。
```
>>> print(hex(0xffffbe8c44c4c08c & ~0b1111))
0xffffbe8c44c4c080
```

最後，我們只需把這個值寫到 cmd.exe 的 Token 值就大功告成了! 這裡使用 ```eq <address> <value>``` 來改變 cmd.exe 的 EPROCESS.Token 欄位的值，然後讓 VM 繼續跑:
```
kd> eq ffffe60fced33080+4b8 ffffbe8c44c4c080
kd> g
```
完成以後，再次在 CMD 中輸入 ```whoami```:
```
C:\Users\seantzou>whoami
nt authority/system
```

可以看到，我們有 system 權限了，提權成功! 🎉🎉🎉

但是，剛剛的 EPROCESS 指標是 kd "送"給我們的，那我們要如何自己找到 cmd.exe 和 system 的 EPROCESS 指標呢?

這裡就來介紹另外兩個結構，**KPCR**、**KPCRB** 和 **KTHREAD**。

* ### KPCR (Kernel Processor Control Region)

    KPCR 是對應到 CPU 的一個結構，就和剛剛講的 EPROCESS 跟程序的關係差不多，結構如下:
    ```c++
    typedef struct _KPCR {
        _NT_TIB NtTib;                                      // +0x000
        _KGDTENTRY64*       GdtBase;                        // +0x000
        _KTSS64*            TssBase;                        // +0x008
        ULONG64             UserRsp;                        // +0x010
        _KPCR*              Self;                           // +0x018
        _KPRCB*             CurrentPrcb;                    // +0x020
        _KSPIN_LOCK_QUEUE*  LockArray;                      // +0x028
        PVOID               Used_Self;                      // +0x030
        _KIDTENTRY64*       IdtBase;                        // +0x038
        ULONG64             Unused[2];                      // +0x040
        UCHAR               Irql;                           // +0x050
        UCHAR               SecondLevelCacheAssociativity;  // +0x051
        UCHAR               ObsoleteNumber;                 // +0x052
        UCHAR               Fill0;                          // +0x053
        ULONG               Unused0[3];                     // +0x054
        USHORT              MajorVersion;                   // +0x060
        USHORT              MinorVersion;                   // +0x062
        ULONG               StallScaleFactor;               // +0x064
        PVOID               Unused1[3];                     // +0x068
        ULONG               KernelReserved[15];             // +0x080
        ULONG               SecondLevelCacheSize;           // +0x0bc
        ULONG               HalReserved[16];                // +0x0c0
        ULONG               Unused2;                        // +0x100
        PVOID               KdVersionBlock;                 // +0x108
        PVOID               Unused3;                        // +0x110
        ULONG               PcrAlign1[24];                  // +0x118
        _KPRCB              Prcb;                           // +0x180
    } KPCR;
    ```
    在64位元系統中，分段暫存器 gs:[0] 即儲存了這個結構，所以我們可以使用 gs:[...] 來讀取 KPCR 結構的欄位。
    而 KPCR 中有一個拓展欄位 ```Prcb``` (偏移量 0x180)， 其結構即為 ```KPRCB```。
    
* ### KPRCB (Kernel Processor Control Block):

    KPRCB 其也儲存有關 CPU 和執行緒的資料。KPRCB 的**部份**結構如下: 
    ```c++
    typedef struct _KPRCB {
        ULONG           MxCsr                      // +0x000 
        UCHAR           LegacyNumber               // +0x004 
        UCHAR           ReservedMustBeZero         // +0x005 
        UCHAR           InterruptRequest           // +0x006 
        UCHAR           IdleHalt                   // +0x007 
        _KTHREAD*       CurrentThread              // +0x008 
        _KTHREAD*       NextThread                 // +0x010 
        _KTHREAD*       IdleThread                 // +0x018 
        UCHAR           NestingLevel               // +0x020 
        UCHAR           ClockOwner                 // +0x021 
        UCHAR           PendingTickFlags           // +0x022 
        
        //[...] 省略
    } KPRCB;
    ```
    
    注意他的 ```CurrentThread```欄位 (偏移量 0x8)，他會指向目前程序執行緒的 ```ETHREAD``` 結構。
    
* ### KTHREAD:

    KTHREAD 簡單來說就是 ETHREAD 結構的內核部分，**部份**結構如下:
    ```c++
    typedef struct _KTHREAD {
        _DISPATCHER_HEADER  Header                    // +0x000 
        PVOID          SListFaultAddress;             // +0x018 
        ULONG64        QuantumTarget;                 // +0x020 
        PVOID          InitialStack;                  // +0x028 
  
        //[...] 省略
        UCHAR               SpecCtrl;                 // +0x07f 
        ULONG               SystemCallNumber;         // +0x080 
        ULONG               ReadyTime;                // +0x084 
        PVOID               FirstArgument;            // +0x088 
        Ptr64 _KTRAP_FRAME  TrapFrame;                // +0x090 
        union {
            _KAPC_STATE     ApcState;                 // +0x098 
            UCHAR           ApcStateFill[43];         // +0x098
        }
        CHAR                Priority;                 // +0x0c3 
        ULONG               UserIdealProcessor;       // +0x0c4 
   
        //[...] 省略
   } KTHREAD;
    ```
    
    他有一個 union, 裡面有一個 ```ApcState```欄位，為 KAPC_STATE 結構。
    ```c++
    typedef struct _KAPC_STATE{
        _LIST_ENTRY ApcListHead[2];             // +0x000
        _KPROCESS*  Process;                    // +0x020
        UCHAR       InProgressFlags;            // +0x028
        UCHAR       KernelApcInProgress : 1;    // +0x028
        UCHAR       SpecialApcInProgress : 1;   // +0x028
        UCHAR       KernelApcPending;           // +0x029
        UCHAR       UserApcPendingAll;          // +0x02a
        UCHAR       SpecialUserApcPending : 1;  // +0x02a
        UCHAR       UserApcPending : 1;         // +0x02a
    } KAPC_STATE;
    ```
    而 ```Process``` 欄位就是其 ```_EPROCESS``` 中 ```Pcb``` 的指標了 !
    
    我們來整理一下整個流程，然後開始寫 shell code:
    
    ![ref5](https://lompandi.github.io/posts/post3/imgs/fetching_eprocess.png)
    
    1. 由讀取 gs:[0x188] (計算過程```Prcb 的偏移量 + CurrentThread 欄位的偏移量 = 0 + 0x180 + 0x8 = 0x188```) 得到 CurrentThread 的指標。
    2. 由讀取 CurrentThread + 0xb8 ```ApcState 偏移量 + Process 偏移量 = 0x98 + 0x20 = 0xb8``` 得到 Process 指標，並且由其找到 _EPROCESS 中的 Pcb 欄位 (偏移量 0x0)，就可以找到 _EPROCESS 的位址了。    
    
    上述流程轉換成組合語言如下:
    
    ```
    mov rdx, gs:[0x188]     ; 0x180 + 0x8 = 0x188 -> CurrentThread pointer
    mov rdx, [rdx + 0xb8]   ; 0x98 + 0x20 = 0xb8  -> Process Pointer
    ```
    
    但是，這個 _EPROCESS 結構是目前執行緒的，我們目前還是不知道 system 和 cmd.exe 的 _EPROCESS 結構啊?
    
    沒關係，我們再看一下 _EPROCESS 結構: 
    ```
    kd> dt nt!_EPROCESS
       +0x000 Pcb              : _KPROCESS
       +0x438 ProcessLock      : _EX_PUSH_LOCK
       +0x440 UniqueProcessId  : Ptr64 Void
       +0x448 ActiveProcessLinks : _LIST_ENTRY
       +0x458 RundownProtect   : _EX_RUNDOWN_REF
       +0x460 Flags2           : Uint4B
       [...]
       +0x4b8 Token            : _EX_FAST_REF
    ```
    
    _EPROCESS 中有一個 ```ActiveProcessLinks```，他是一個 doubly linked list 的節點 _LIST_ENTRY，結構很簡單:
    ```c++
    typedef struct _LIST_ENTRY{
        _LIST_ENTRY* Flink;
        _LIST_ENTRY* Blink;
    } LIST_ENTRY;
    ```
    整個串列長得像這樣:
    
    ![ref6](https://lompandi.github.io/posts/post3/imgs/ActiveProcessLink.png)
    
    在這裡 Flink 會指向下一個 EPROCESS 的 ActiveProcessLinks 的 Flink，Blink 則指向上一個 EPROCESS 的 ActiveProcessLinks 的 Flink，
    由節點切入可以走訪所有程序的 _EPROCESS 結構，我們就可以利用這個串列走訪所有 _EPROCESS 結構。
    
    而 _EPROCESS 的另外一個欄位 UniqueProcessId (PID) 位址即為 ```Flink 欄位的位址 - 8```，這代表如果我們讀取每個 Flink 的位址 - 8 的值，就可以一邊走一邊比對 PID 來找尋 system 和 cmd.exe 的 _EPROCESS 結構了。

    接下來，我們先著手寫模擬的 shellcode 吧，
    
    當然，Kernel 做為一個系統中重要的物件，自然不會乖乖站在那邊給你打，Kernel Mode 有一套有別於 User Mode     的保護，最常見的有下列幾項:

* ### SMAP (Supervisor Mode Access Prevention)
由控制暫存 Cr4 中的第 20 個位元控制。若開啟，將**禁止核心模式的程序直接存取所有使用者模式分區的資料**。

* ### SMEP (Supervisor Mode Execution Prevention)
由控制暫存 Cr4 中的第 20 個位元控制。若開啟，將**禁止核心模式的程序執行位於使用者模式分區的代碼**。

* ### KVAS (Kernel Virtual Address Shadow)




