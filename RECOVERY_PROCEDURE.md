### 🔄 Interactive Recovery Procedure (amrecover)

Use this method for granular file recovery or when an interactive interface is preferred over `amrestore`.

1. **Initialize Profile:** `amrecover [Set_Name]`
2. **Set Target Directory:** `lcd /path/to/restore` (Ensure disk space > backup size)
3. **Target Selection:**
   * `sethost [hostname]`
   * `setdisk [disk_device]`
   * `setdate YYYY-MM-DD`
4. **Media Selection:** `setdevice hp-robot` (Utilizes the configured tape library)
5. **File Marking:**
   * `ls` (List files in the dump)
   * `add [filename]` (Add to extraction list)
6. **Execution:** `extract`
   * *Note: AMANDA will automatically command the autoloader to mount the required volume.*