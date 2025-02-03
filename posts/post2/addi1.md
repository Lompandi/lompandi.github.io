### C++ Calling Convention

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

而一般來說會根據這個參數傳入規則來規範大致的呼叫慣例:
1. ```__fastcall``: 參數較少時，可以宣告函數參數不使用堆疊。

2. ```__cdecl```: C/C++ 編譯器使用的預設呼叫慣例，同時也是 C 標準的一部份。

它代表“C Declaration”，定義函數如何從呼叫者接收參數以及函數呼叫後如何清理堆疊。

- 參數傳遞：參數會依照從右到左的順序壓入堆疊（表示最後一個參數先 push）。

- 堆疊清理：呼叫者負責在函數返回後清理堆疊。這意味著呼叫者需要在函數呼叫後調整堆疊指標（透過從堆疊中刪除參數）。

3. ```__stdcall```: 與 ```__cdecl``` 類似，但**被呼叫者**清理堆疊，參數從右向左傳遞。這在 Windows API 函數中常用。

知道這些後，我們來看 IDA 反組譯後的 ```Human::get_legs```:
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

這裡我們看到 ```get_legs``` 先是壓入基底暫存 (ebp)， 之後把 stack top (esp 代表的位址) 向下移4個 BYTE
以騰出空間給 var_4，之後賦予 var_4 ecx(即第一個參數) 的值，然後把2放入 eax，接著 return。

看到這裡可能會有人問 :

"```get_legs``` 不是沒有參數嗎? 為啥編譯後莫名其妙多出一個參數來了?"

這裡說明一下，C++中類別的成員函數 (Member Function) 如果沒被宣告為靜態 (即使用```static```關鍵字) 的話，編譯後產生的第一個參數將會是```this```指標，且編譯後不論```this```是否在函數中被使用，編譯器都會將其傳入。
