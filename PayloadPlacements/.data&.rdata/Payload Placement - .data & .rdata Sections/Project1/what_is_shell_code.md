# Shellcode and PE Section Placement - Complete Study Notes

## 1. What is Shellcode?

Shellcode is position-independent machine code that's typically used as a payload in exploits. In this case, we are looking at Windows x64 shellcode that launches calc.exe.

Arrays usually contain raw assembly instructions in hexadecimal format. When executed, these bytes perform specific operations.

**Key Concept**: The hex bytes you see are the actual machine code (assembled instructions). They're not "turned into" assembly - they already ARE the CPU instructions in their raw binary form.

```
Source Code (ASM) → Assembler → Machine Code (Hex Bytes) → CPU Executes
```

### Why Shellcode Exists
- **Exploit Constraints**: Buffer overflows give limited space and no normal program environment
- **No Dependencies**: Can't rely on imports, libraries, or fixed addresses
- **Stealth Requirements**: Must work without leaving obvious traces
- **Position Independence**: Must work at any memory address

## 2. How Does a Hacker Get Shellcode?

### A. Using Shellcode Generators (Most Common)

**Metasploit's msfvenom:**
```bash
msfvenom -p windows/x64/exec CMD=calc.exe -f c
```
- `-p` = payload type
- `CMD=calc.exe` = command to execute
- `-f c` = format as C array

### B. The Build Process: From C to Shellcode

**Step 1: Start with Normal C Code**
```c
// What we want to accomplish
#include <windows.h>
int main() {
    WinExec("calc.exe", 1);
    ExitProcess(0);
}
```

**Step 2: Problem - This Won't Work as Shellcode!**
- Requires kernel32.dll to be loaded
- Needs Import Address Table (IAT)
- Depends on fixed memory addresses
- Contains null bytes (0x00) that break string operations

**Step 3: Rewrite in Assembly with Special Techniques**
- PEB walking to find APIs
- API hashing instead of strings
- Position-independent code
- Null byte avoidance

## 3. Shellcode Execution Phases

### Phase 1: Stack Alignment & Setup (First few bytes)
```
0xFC = CLD (Clear Direction Flag)
0x48, 0x83, 0xE4, 0xF0 = AND RSP, -16 (Aligns stack to 16-byte boundary)
```
**Why?** Windows expects aligned stack for API calls

### Phase 2: PEB Walking (Process Environment Block)
The shellcode uses a technique called "PEB walking" to find Windows API functions without hardcoding addresses:
- Accesses PEB through `GS:[0x60]` (that's what bytes around offset 0x16 do)
- Walks through loaded modules to find kernel32.dll
- Searches for specific API functions by hash

**Why PEB Walking?**
- Always at predictable offset from GS register
- Contains list of all loaded modules
- No hardcoded addresses needed
- Works on all Windows versions

### Phase 3: API Resolution
The shellcode implements a custom hash-based API resolver to find functions like:
- **WinExec** (to execute calc.exe)
- **ExitProcess** (to cleanly exit)

**API Hashing Example:**
```c
// Instead of storing "WinExec" (8 bytes + null)
// Store hash: 0x876F8B31 (only 4 bytes!)

// Hash algorithm (ROR13)
DWORD hash = 0;
for (char c : "WinExec") {
    hash = (hash >> 13) | (hash << 19);  // ROR 13
    hash += c;
}
```

### Phase 4: Execution
At the end, you can see:
```
0x63, 0x61, 0x6C, 0x63, 0x00  // "calc\0" in ASCII
```
This is the command string passed to WinExec.

## 4. PE Section Placement

From your PDF notes, this code demonstrates:

### .data section: Where Data_RawData is stored (without const)
- **Permissions**: Read-Write (RW)
- **Runtime**: Can be modified at runtime
- **Use Case**: Good for encrypted payloads that need decryption
- **Example Address**: 0x00007FF7B7603000 (from PDF)

**Built for:**
- Encrypted shellcode that self-decrypts
- Polymorphic code that changes itself
- Multi-stage payloads

### .rdata section: Where const variables are stored
- **Permissions**: Read-only (R)
- **Runtime**: Cannot be modified at runtime
- **Use Case**: Better for static payloads
- **Access**: Any modification attempts cause access violations

**Built for:**
- Simple, static payloads
- When size matters more than stealth
- Quick deployment scenarios

## 5. Anti-Malware Implications

As a red team student, you should understand:

### Detection Challenges:
- Shellcode in `.data` can be encrypted/obfuscated
- `.rdata` shellcode is static and easier to signature
- Dynamic analysis catches behavior regardless of storage

### Memory Permissions:
- `.data` is RW but not executable by default (DEP/NX)
- Would need `VirtualProtect()` or similar to make executable

**Required Steps to Execute:**
```c
// 1. Change memory permissions
VirtualProtect(Data_RawData, sizeof(Data_RawData), 
               PAGE_EXECUTE_READWRITE, &oldProtect);

// 2. Cast to function pointer
void (*shellcode)() = (void(*)())Data_RawData;

// 3. Execute
shellcode();
```

### Analysis Tools:
- The PDF shows using `dumpbin.exe` to examine PE sections
- xdbg shows runtime memory layout and protections
- Static analysis can identify shellcode patterns
- Dynamic analysis monitors behavior

## 6. Why These Techniques?

### The Complete Picture
```
Developer Intent → C Code → Assembly (with constraints) → Machine Code → PE Section
                                    ↑                           ↑
                            Position Independence        Memory Protections
                            No Null Bytes                Section Properties
                            API Resolution               Runtime Behavior
```

### Why This Complexity?
1. **Environment Constraints**: Exploits provide hostile environments
2. **Detection Evasion**: Each technique bypasses specific defenses
3. **Reliability**: Must work across different Windows versions
4. **Size Limitations**: Buffer overflows have size constraints

### Key Takeaways
- **Understand Section Properties**: Know where your payload will be stored and what permissions it will have
- **Every Byte Matters**: Size and content restrictions shape shellcode design
- **It's Engineering**: Shellcode is code engineered within extreme constraints
- **Constant Evolution**: New defenses require new techniques