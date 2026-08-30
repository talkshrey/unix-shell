# Unix Shell

## 📋 Brief Description

**tsh** is a lightweight Unix shell implementation that provides essential job control and command execution capabilities. It allows users to execute programs in the foreground or background, manage running processes, and handle signals for job control.

### Key Features
- **Foreground/Background Execution**: Run jobs in foreground (interactive) or background (non-interactive) modes
- **Job Management**: Track and manage multiple concurrent jobs with unique job IDs
- **Signal Handling**: Handle SIGINT (Ctrl-C), SIGTSTP (Ctrl-Z), and SIGCHLD signals
- **Built-in Commands**: `quit`, `jobs`, `fg` (foreground), and `bg` (background)
- **I/O Redirection**: Support input (`<`) and output (`>`) redirection
- **Process Control**: Stop, resume, and terminate jobs

---

## 🔄 Shell Execution Flow

```mermaid
graph TD
    A["User Input\n(Command Line)"] --> B["Read Command\n(fgets)"]
    B --> C["Parse Command\n(parseline)"]
    C --> D{Command Type?}
    D -->|Built-in| E["Handle Built-in\n(quit/jobs/fg/bg)"]
    D -->|External Program| F["Create Child Process\n(fork)"]
    E --> Z["Return to Prompt"]
    F --> G["Setup I/O Redirection\nInput/Output Files"]
    G --> H["Execute Program\n(execve)"]
    H --> I{Job Type?}
    I -->|Foreground| J["Wait for Job\n(sigsuspend)"]
    I -->|Background| K["Print Job ID & PID\nContinue Shell"]
    J --> Z
    K --> Z
```

---

## 🎛️ Job States & Transitions

```mermaid
stateDiagram-v2
    [*] --> FG: Fork new process
    [*] --> BG: Fork with &

    FG --> ST: Ctrl-Z (SIGTSTP)
    BG --> ST: Ctrl-Z (SIGTSTP)
    
    ST --> FG: fg command\n+ SIGCONT
    ST --> BG: bg command\n+ SIGCONT
    
    FG --> [*]: Exit/Terminate
    BG --> [*]: Exit/Terminate
    ST --> [*]: Exit/Terminate

    note right of FG
        Shell waits for 
        job completion
    end note
    
    note right of BG
        Shell continues
        accepting input
    end note
    
    note right of ST
        Job suspended
        in memory
    end note
```

---

## 🔐 Signal Handling Architecture

```mermaid
graph LR
    A["SIGCHLD\n(Child Terminated)"] --> B["sigchld_handler"]
    C["SIGINT\n(Ctrl-C)"] --> D["sigint_handler"]
    E["SIGTSTP\n(Ctrl-Z)"] --> F["sigtstp_handler"]
    
    B --> B1["waitpid -1 WNOHANG"]
    B1 --> B2{"Check Status"}
    B2 -->|Exited| B3["delete_job"]
    B2 -->|Signaled| B4["Print Termination\ndelete_job"]
    B2 -->|Stopped| B5["Print Stopped Msg\njob_set_state ST"]
    
    D --> D1["Get FG Job PID"]
    D1 --> D2["kill -FG_PID SIGINT"]
    
    F --> F1["Get FG Job PID"]
    F1 --> F2["kill -FG_PID SIGTSTP"]
```

---

## 📊 Built-in Commands Summary

| Command | Purpose | Behavior |
|---------|---------|----------|
| `quit` | Exit shell | Terminates the shell process |
| `jobs [> outfile]` | List jobs | Shows all active jobs with ID, PID, and state |
| `fg <jid\|pid>` | Run in foreground | Move stopped job to foreground, resume with SIGCONT |
| `bg <jid\|pid>` | Run in background | Move stopped job to background, resume with SIGCONT |

---

## ⚙️ Core Components

### 1. **Command Parser** (`parseline`)
- Parses command line into tokens
- Detects foreground/background execution (`&` suffix)
- Handles I/O redirection (`<`, `>`)
- Supports quoted arguments

### 2. **Job List Manager** (`tsh_helper.c`)
- Maintains list of up to 64 concurrent jobs
- Tracks Job ID (JID), PID, state, and command line
- Provides async-signal-safe operations

### 3. **Signal Handlers**
- **SIGCHLD**: Reaps terminated child processes
- **SIGINT**: Forwards Ctrl-C to foreground job
- **SIGTSTP**: Forwards Ctrl-Z to foreground job
- **SIGQUIT**: Emergency cleanup on quit signal

### 4. **Process Management**
- Uses `fork()` to create child processes
- Uses `execve()` to execute external programs
- Uses `waitpid()` with `WNOHANG | WUNTRACED` for non-blocking reaping
- Manages process groups for signal delivery

---

## 🔗 Data Flow: Command Execution

```mermaid
graph TD
    A["eval cmdline"] --> B["parseline"]
    B --> C["struct cmdline_tokens"]
    C --> D{Builtin?}
    D -->|Yes| E["Handle: quit/jobs/fg/bg"]
    D -->|No| F["Block Signals"]
    F --> G["fork"]
    G --> H{Child or Parent?}
    H -->|Child| I["Setup I/O"]
    I --> J["setpgid 0,0\nUnblock Signals"]
    J --> K["execve"]
    H -->|Parent| L["add_job"]
    L --> M{FG or BG?}
    M -->|FG| N["sigsuspend\nWait for FG completion"]
    M -->|BG| O["Print Job Info"]
    N --> P["Unblock Signals\nReturn"]
    O --> P
```

---

## 🛡️ Signal Masking Strategy

The shell uses signal masking to prevent race conditions:

```mermaid
graph LR
    A["Before fork"] --> B["sigprocmask\nBLOCK: SIGCHLD, SIGINT, SIGTSTP"]
    B --> C["fork"]
    C --> D["Child Process"]
    D --> E["sigprocmask\nUNBLOCK signals"]
    E --> F["execve"]
    C --> G["Parent Process"]
    G --> H["add_job"]
    H --> I["Manage with signals blocked"]
    I --> J["sigprocmask\nUNBLOCK signals"]
```

---

## 📝 Job Control Example

```
tsh> ./program &           ← Background execution
[1] (12345) ./program      ← Job ID [1], PID 12345
tsh> ./another             ← Foreground execution
^Z                         ← User presses Ctrl-Z
Job [2] (12346) stopped by signal 20
tsh> jobs                  ← List active jobs
[1] (12345) Running     ./program
[2] (12346) Stopped     ./another
tsh> fg %2                 ← Resume job 2 in foreground
tsh> bg %2                 ← Resume job 2 in background
[2] (12346) ./another
```

---

## 🏗️ Architecture Layers

```mermaid
graph TB
    subgraph User["User Interaction"]
        A["Shell Prompt"]
        B["Command Input"]
    end
    
    subgraph Core["Shell Core"]
        C["eval - Command Evaluator"]
        D["Signal Handlers"]
        E["Job List Manager"]
    end
    
    subgraph System["System Layer"]
        F["fork/execve/waitpid"]
        G["Signal Handling\nSIGCHLD, SIGINT, SIGTSTP"]
        H["Process Groups"]
    end
    
    User --> Core
    Core --> System
    System --> Kernel["Linux Kernel"]
    
    style User fill:#e1f5ff
    style Core fill:#f3e5f5
    style System fill:#fff3e0
    style Kernel fill:#ede7f6
```

---

## 🔧 Implementation Language

- **Primary**: C (CSC 15-213 Shell Lab)
- **Helper Libraries**: CSAPP (Computer Systems: A Programmer's Perspective)
- **Compilation**: GCC with strict error checking
- **Dependencies**: POSIX-compliant system calls

---

## 📌 Key Design Principles

1. **Async-Signal Safety**: All signal handlers use only async-signal-safe functions
2. **Race Condition Prevention**: Strategic signal blocking around critical sections
3. **Resource Management**: Proper cleanup of job list and file descriptors
4. **Error Handling**: Graceful error messages for permission denied and missing files
5. **Job ID Recycling**: Efficient reuse of job IDs (1-64)

---

## Notes
Upon request, code can shown for professional purposes.
