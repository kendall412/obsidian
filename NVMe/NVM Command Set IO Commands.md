
For the **NVMe NVM Command Set Specification v1.2**, the standard **NVM I/O commands** are:

| Opcode | Command | Data direction |
|---:|---|---|
| `00h` | **Flush** | No data transfer |
| `01h` | **Write** | Host to controller |
| `02h` | **Read** | Controller to host |
| `04h` | **Write Uncorrectable** | No data transfer |
| `05h` | **Compare** | Host to controller |
| `08h` | **Write Zeroes** | No data transfer |
| `09h` | **Dataset Management** | Host to controller |
| `0Ch` | **Verify** | No data transfer |
| `0Dh` | **Reservation Register** | Host to controller |
| `0Eh` | **Reservation Report** | Controller to host |
| `11h` | **Reservation Acquire** | Host to controller |
| `15h` | **Reservation Release** | Host to controller |
| `19h` | **Copy** | Host to controller |

So the full list is:

1. Flush  
2. Write  
3. Read  
4. Write Uncorrectable  
5. Compare  
6. Write Zeroes  
7. [[DSM (Dataset Management)]]  
8. Verify  
9. Reservation Register  
10. Reservation Report  
11. Reservation Acquire  
12. Reservation Release  
13. Copy  

Note: this list is for the **NVM Command Set**. Other NVMe I/O command sets, such as **Zoned Namespace**, **Key Value**, or **Computational Programs**, have their own separate command-set specifications.

