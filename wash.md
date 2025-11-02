# 🧺 Washing Machine Socket Project — Simplified Design & Roadmap

A simple simulation project using **C**, **sockets**, **timerfd**, **select()**, and a **linked list queue**. Ideal for beginners building their first client-server simulation.

---

## 📘 Project Overview

You are building a **Washing Machine Server** that manages multiple washing machines and multiple clients over sockets. Only a limited number of clients can wash simultaneously. Others must wait in a queue until a machine becomes free.

**Key features:**

* Multiple clients connect via TCP sockets.
* Each client sends a `WASH <SPIN_OPTION>` command.
* The server manages a fixed number of washing machines.
* If all machines are busy, clients are queued using a linked list.
* Each wash uses a **timerfd** to represent the washing duration.
* When a wash completes, the next client from the queue is assigned automatically.

---

## ⚙️ Components

### 🖥️ Server

* Handles multiple client connections.
* Manages the washing machines.
* Uses a linked list as a queue for waiting clients.
* Uses `select()` to monitor socket and timer events.

### 💡 Client

* Connects to the server via socket.
* Sends washing commands.
* Waits for updates (wash started / completed / queued).

### 🧩 Linked List (Pre-built)

* Your DLL/SLL library will be used to implement the waiting queue.
* Supports enqueue and dequeue operations.

---

## 🧱 System Design

```
CLIENT1 ---> SOCKET --->|
CLIENT2 ---> SOCKET --->|--> SERVER
CLIENT3 ---> SOCKET --->|     |
                         |     +--> MACHINE[0..N-1]
                         |     +--> QUEUE (Linked List)
                         |     +--> TIMERFD + SELECT()
```

---

## 🧰 Core Structures

### Machine

```c
typedef struct {
    int id;             // Machine ID
    int busy;           // 0 = free, 1 = busy
    int timerfd;        // Timer for wash cycle
    int client_fd;      // Socket of client currently using it
    int spin_time;      // Wash duration (seconds)
} Machine;
```

### Client Job (for Queue)

```c
typedef struct {
    int client_fd;      // Socket descriptor of waiting client
    int spin_time;      // Requested wash time
} ClientJob;
```

---

## ⏱️ Spin Options

| Option | Duration (Seconds) |
| ------ | ------------------ |
| QUICK  | 5                  |
| NORMAL | 10                 |
| HEAVY  | 20                 |

```c
typedef struct {
    char name[16];
    int duration;
} SpinOption;

SpinOption spins[] = {
    {"QUICK", 5},
    {"NORMAL", 10},
    {"HEAVY", 20}
};
```

---

## 🧩 Functional Flow

```
Client sends → WASH NORMAL
        ↓
Server checks free machines
        ↓
Free → start timerfd, send "Your wash started"
Busy → enqueue client, send "All machines busy, queued"
        ↓
Timer expires → send "Wash complete" → assign next queued client
```

---

## 🧠 Server Logic Flow

1. **Initialize washing machines**

   ```c
   for (int i = 0; i < N; i++) {
       machines[i].id = i + 1;
       machines[i].busy = 0;
       machines[i].timerfd = timerfd_create(CLOCK_MONOTONIC, 0);
   }
   ```

2. **Create TCP Server Socket**

   ```c
   sockfd = socket(AF_INET, SOCK_STREAM, 0);
   bind(sockfd, ...);
   listen(sockfd, 5);
   ```

3. **Use select() to monitor**

   * Listening socket → for new connections
   * Client sockets → for messages
   * Timerfds → for wash completion

4. **Client Command Example**

   ```
   WASH QUICK
   ```

   * Parse spin type.
   * Find matching spin duration.
   * Assign to free machine or enqueue if all busy.

5. **Start Machine Timer**

   ```c
   struct itimerspec its = {0};
   its.it_value.tv_sec = spin_duration;
   timerfd_settime(machine.timerfd, 0, &its, NULL);
   ```

6. **When Timer Expires**

   * Read timerfd.
   * Notify client wash completed.
   * Free the machine.
   * Check queue and start next waiting client.

---

## 💬 Client Program

1. Connect to server:

   ```c
   sock = socket(AF_INET, SOCK_STREAM, 0);
   connect(sock, ...);
   ```

2. Send command:

   ```c
   char spin[20];
   printf("Enter spin option (QUICK/NORMAL/HEAVY): ");
   scanf("%s", spin);
   sprintf(msg, "WASH %s", spin);
   send(sock, msg, strlen(msg), 0);
   ```

3. Wait for messages:

   ```c
   while (recv(sock, msg, sizeof(msg), 0) > 0) {
       printf("%s\n", msg);
   }
   ```

---

## 🧩 File Structure

```
project/
│
├─ server.c         # Handles washing logic, sockets, select()
├─ client.c         # Sends commands and receives messages
├─ linkedlist.c     # Your queue implementation
├─ linkedlist.h
└─ Makefile
```

---

## 🪜 Step-by-Step Roadmap

### PHASE 1 — Setup Base

* Create server socket and accept multiple clients using `select()`.

### PHASE 2 — Add Machine Logic

* Initialize washing machine array.
* Assign clients to machines.

### PHASE 3 — Add Timer Functionality

* Use `timerfd` to simulate wash durations.

### PHASE 4 — Add Queue System

* Integrate linked list to hold waiting clients.

### PHASE 5 — Client Communication

* Clients send `WASH <spin>` commands.
* Server sends status updates.

### PHASE 6 — Final Cleanup

* Handle disconnections.
* Close sockets and free memory.

---

## 🧩 Example Output

**Server Console:**

```
[Server] Machine 1 started washing Client 2 (NORMAL, 10s)
[Server] Machine 2 started washing Client 3 (HEAVY, 20s)
[Server] All machines busy. Client 4 queued.
[Server] Machine 1 finished Client 2.
[Server] Machine 1 started washing Client 4 (QUICK, 5s).
```

**Client 4 Console:**

```
All machines busy. You are in queue.
Your wash started on machine 1 for 5 seconds.
Your wash completed!
```

---

## ✅ Design Choices (Simplified for Beginners)

| Area          | Choice      | Reason                               |
| ------------- | ----------- | ------------------------------------ |
| Concurrency   | select()    | Easiest multi-FD management          |
| Queue         | Linked list | Simple FIFO and already available    |
| Timing        | timerfd     | No threads, pure event-based control |
| Communication | TCP sockets | Works for multiple clients           |
| Complexity    | Low         | Beginner-friendly, minimal setup     |

---

## 🧠 Tips for Success

* Always `read()` the `timerfd` when it expires (to reset it).
* Check all return values (`socket`, `bind`, `select`, `read`, `write`).
* Use print statements to trace logic step-by-step.
* Test with multiple terminal clients.
* Keep durations short (5–20s) for quick testing.

---

## 🧪 Testing Checklist

* [ ] Multiple clients can connect simultaneously.
* [ ] Only N clients can wash at once.
* [ ] Queued clients start automatically when machines free up.
* [ ] Wash duration matches spin type.
* [ ] Clients receive correct messages.
* [ ] Server handles client disconnects safely.

---

## 🧩 Future Enhancements

* Add more operations: Rinse, Dry.
* Add logging or status display.
* Add configuration file for spin times.
* Add color-coded terminal outputs.

---

## 🧾 License

Use freely for learning and educational purposes.
