# Complete Shellcode & PE Sections Study Guide

## 1. Understanding Shellcode Fundamentals

### What is Shellcode?
Shellcode is position-independent machine code typically used as a payload in exploits. In our examples, we're examining Windows x64 shellcode that launches calc.exe.

**Critical Concept**: The hex bytes (0xFC, 0x48, etc.) ARE the actual machine code - they're not "turned into" assembly, they already ARE the CPU instructions in raw binary form!

```
Source Code (ASM) → Assembler → Machine Code (Hex Bytes) → CPU Executes
```

### Why Shellcode Exists
- **Exploit Constraints**: Buffer overflows provide limited space with no normal program environment
- **No Dependencies**: Cannot rely on imports, libraries, or fixed addresses
- **Position Independence**: Must work at any memory address
- **Stealth Requirements**: Must operate without obvious traces

## 2. How Attackers Create Shellcode

### Method 1: Shellcode Generators (Most Common)
```bash
msfvenom -p windows/x64/exec CMD=calc.exe -f c
```
- Instant generation of common payloads
- Multiple output formats
- Built-in encoders for evasion

### Method 2: Custom Development Process

**Step 1: Start with C Code**
```c
// Goal: Execute calc.exe
#include <windows.h>
int main() {
    WinExec("calc.exe", 1);
    ExitProcess(0);
}
```

**Step 2: Identify Problems**
- Requires kernel32.dll imports
- Uses Import Address Table (IAT)
- Contains null bytes (0x00)
- Not position-independent

**Step 3: Rewrite in Assembly with Special Techniques**
- PEB walking for API resolution
- API hashing instead of strings
- Stack-based string construction
- Null byte avoidance

## 3. Shellcode Execution Phases Explained

### Phase 1: Stack Alignment & Setup
```
0xFC = CLD (Clear Direction Flag)
0x48, 0x83, 0xE4, 0xF0 = AND RSP, -16 (Aligns stack to 16-byte boundary)
```
**Why?** Windows x64 ABI requires 16-byte stack alignment for API calls

### Phase 2: PEB Walking (Process Environment Block)
```asm
mov rax, [gs:0x60]    ; Get PEB address
mov rax, [rax+0x18]   ; PEB->Ldr
mov rax, [rax+0x20]   ; InMemoryOrderModuleList
```

**Purpose**: Find kernel32.dll without hardcoded addresses
- GS:[0x60] always points to PEB
- Walk module list to find kernel32.dll
- Parse PE headers to find exports

### Phase 3: API Resolution via Hashing
```c
// Instead of storing "WinExec" (8 bytes)
// Use ROR13 hash: 0x876F8B31 (4 bytes)

for each export:
    compute_hash(export_name)
    if (hash == 0x876F8B31)  // WinExec
        save_function_address
```

### Phase 4: Command Execution
```
0x63, 0x61, 0x6C, 0x63, 0x00  // "calc\0" in ASCII
```
Final shellcode calls: `WinExec("calc", SW_SHOW)`

## 4. PE Section Placement Strategies

### .data Section Approach
```c
// No const qualifier - goes to .data
unsigned char Data_RawData[] = { /* shellcode */ };
```

**Characteristics**:
- **Default Permissions**: RW- (Read, Write, NO Execute)
- **Runtime Behavior**: Can be modified (good for encryption)
- **Memory Address**: e.g., 0x00007FF7B7603000
- **Use Case**: Encrypted/obfuscated payloads

**Execution Requirements**:
```c
// Must add execute permission
VirtualProtect(Data_RawData, sizeof(Data_RawData), 
               PAGE_EXECUTE_READWRITE, &oldProtect);
```

### .rdata Section Approach
```c
// const qualifier - goes to .rdata
const unsigned char Rdata_RawData[] = { /* shellcode */ };
```

**Characteristics**:
- **Default Permissions**: R-- (Read only)
- **Runtime Behavior**: Cannot be modified
- **Access Violations**: Any write attempt crashes
- **Use Case**: Static payloads, smaller file size

### .text Section Approach (Advanced)
```c
#pragma section(".text")
__declspec(allocate(".text")) const unsigned char Text_RawData[] = { /* shellcode */ };
```

**Characteristics**:
- **Default Permissions**: R-X (Read, Execute)
- **Stealthier**: Code in code section looks normal
- **Still Needs**: VirtualProtect for write access (RWX)
- **Use Case**: Evading security tools that only scan data sections

## 5. Practical Execution Examples

### Basic Execution Attempt (Will Fail)
```c
int main() {
    printf("[i] Shellcode at: 0x%p\n", Data_RawData);
    getchar();
    return 0;  // Never executes shellcode!
}
```
**Result**: Program exits normally, shellcode never runs

### DEP/NX Protection Error
```c
// Trying to execute without changing permissions
void (*shellcode)() = (void(*)())Data_RawData;
shellcode();  // Access Violation 0xC0000005!
```
**Result**: Crash - Data Execution Prevention blocks execution

### Correct Execution Method
```c
int main() {
    DWORD oldProtect;
    
    // 1. Make memory executable
    if (!VirtualProtect(Data_RawData, sizeof(Data_RawData), 
                        PAGE_EXECUTE_READWRITE, &oldProtect)) {
        printf("[-] VirtualProtect failed: %d\n", GetLastError());
        return 1;
    }
    
    // 2. Execute shellcode
    printf("[+] Executing shellcode...\n");
    ((void(*)())Data_RawData)();
    
    return 0;  // May not reach here if shellcode calls ExitProcess
}
```

## 6. Security Implications & Detection

### Why Modern Defenses Work
1. **DEP/NX Bit**: Prevents code execution in data sections
2. **ASLR**: Randomizes memory addresses
3. **CFG**: Control Flow Guard detects abnormal execution
4. **Signature Detection**: Known shellcode patterns

### Defender Detection Points
- **Static**: Known shellcode byte patterns
- **Import Analysis**: VirtualProtect usage suspicious
- **Behavioral**: PEB walking, API hashing techniques
- **Memory**: RWX permissions are red flags

### Evasion Techniques (Educational)
```c
// 1. Encryption
for(int i = 0; i < size; i++)
    encrypted[i] ^= 0xAA;

// 2. Delayed execution
Sleep(10000);  // Sandbox evasion

// 3. API obfuscation
GetProcAddress(GetModuleHandle("kernel32"), decrypt("VirtualProtect"));

// 4. Section hopping
memcpy(text_section, data_section, size);
```

## 7. Analysis Tools & Techniques

### Static Analysis
```bash
# Examine PE sections
dumpbin.exe /ALL malware.exe

# Look for:
# - Section characteristics (RWX)
# - Suspicious strings
# - Known patterns
```

### Dynamic Analysis
- **Debugger**: Set breakpoints on VirtualProtect
- **Monitor**: API calls, memory modifications
- **Sandbox**: Observe behavior in isolated environment

### Memory Analysis
```
xdbg showing:
- .data at 0x00007FF7B7603000 (RW-)
- Shellcode at offset +0x40
- Protection changes to RWX
```

## 8. Key Learning Points

### For Red Team
1. **Section Choice Matters**:
   - .data for dynamic/encrypted payloads
   - .text for stealth
   - .rdata when size matters

2. **Bypass Techniques**:
   - Understand memory protections
   - Know detection methods
   - Layer evasion techniques

3. **Execution Flow**:
   - Storage ≠ Execution
   - Permissions must be correct
   - Monitor for detection points

### For Blue Team
1. **Detection Opportunities**:
   - VirtualProtect on data sections
   - PEB access patterns
   - Unusual section characteristics

2. **Analysis Approach**:
   - Static: What's in the file?
   - Dynamic: What does it do?
   - Memory: How does it change?

### The Bottom Line
Understanding shellcode isn't just about the bytes - it's about:
- **Why** each technique exists (constraints drive design)
- **How** protections work (and fail)
- **Where** detection happens (and evasion)
- **What** makes good shellcode (reliability + stealth)

This knowledge helps both in creating secure software and understanding attacks!