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
    uint64_t AsUINT64;
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

首先，將 CR3 轉換為```MMPTE_HARDWARE```，將其 ```PageFrameNumber```  乘以**分頁大小(通常為4096)**，得到**PML4 的基底實體位址**。

再將要轉換的虛擬位址中的 PML4 Index 乘以 8，加上 PML4 的基底實體位址，得到 **PML4E 的實體位址**。


從 PLM4E 的實體位址讀入 8 個位元組並轉換為```MMPTE_HARDWARE```，將其 ```PageFrameNumber``` 乘以 4096，得到 **PDPT 的基底實體位址**。

將要轉換的虛擬位址中的 PDPT Index 乘以 8，加上 PDPT 的基底實體位址，得到 **PDPTE 的實體位址**。


從 PDPTE 的實體位址讀入 8 個位元組並轉換為```MMPTE_HARDWARE```，將其 ```PageFrameNumber``` 乘以 4096，得到 **PD 的基底實體位址**。

這時如果 PDPTE 的 PS 旗標（位元 7）為 1，無須分頁表。直接將 PD 的基底實體位址與虛擬位址中前 30 個 位元相加，**即可直接對應所求的實體記憶體位址**。


如果 PDPTE 的 PS 旗標為 0 ，那麼就像前兩次所做的那樣，將虛擬位址中的 PD Index 乘以 8 個然後加上 **PD 基底實體位址**，得到 **PDE 的實體位址**。


從 PDE 的實體位址讀入 8 個位元組並轉換為```MMPTE_HARDWARE```，將其 ```PageFrameNumber``` 乘以 4096 得到**PT 的基底實體位址**。

如果 PDE 的 PS 旗標為 1，直接將虛擬位址中前 28 個位元和 **PT 基底實體位址** 相加，**即可直接對應所求的實體記憶體位址**。


反之，如果 PDE 的 PS 旗標為 0，則將虛擬位址中的 PT Index 乘以 8 個然後加上 **PT 基底實體位址**，得到  **PTE 的實體位址**。之後，

從 PTE 的實體位址讀入 8 個位元組並轉換為```MMPTE_HARDWARE```，將其 ```PageFrameNumber``` 乘以 4096 加上虛擬位址的 Offset，**即可得到所求的實體記憶體位址**。

有分支計算的部份可能有些眼花撩亂，幫大家整理一下:

#### 1. *if* PDPTE.PS is *1* -> *Physical Address* = *PD Base* + *Virtual Address*[0:29]
#### 2. *if* PDE.PS is *1* ->   *Physical Address* = *PT Vase* + *Virtual Address*[0:27]

幸運的是，我們不必手動計算每個分頁結構表，因為 kd 中的!pte指令可以直接幫我們計算。

```
kd> !pte fffff804`72400000
                                           VA fffff80472400000
PXE at FFFFDAED76BB5F80    PPE at FFFFDAED76BF0088    PDE at FFFFDAED7E011C90    PTE at FFFFDAFC02392000
contains 0000000003F09063  contains 0000000003F0A063  contains 8A000000020001A1  contains 0000000000000000
pfn 3f09      ---DA--KWEV  pfn 3f0a      ---DA--KWEV  pfn 2000      -GL-A--KR-V  LARGE PAGE pfn 2000
```

其中每條虛線對應一個特定的控制位元。PXE 部分實際上是指該線性位址的 PML4 表，而 PPE 則表示分頁 PDP 表。(為甚麼要亂給人家改名字??😭)
