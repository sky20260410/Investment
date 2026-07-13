# basepri

basepri

例如：我如何监控是哪个函数操作了 basepri 寄存器                                                                                                                                                                    

⏺ 要监控哪个函数操作了 BASEPRI 寄存器，你可以使用以下几种方法：                                                                                                                                                      

⏺ 方法1：使用 SWO 追踪和 ITM 仪器化                                                                                                                                                                                  

  步骤：                                                                                                                                                                                                             

  1. 启用 SWO 追踪                                                                                                                                                                                                   
  // 在系统初始化时添加                                                                                                                                                                                              
  CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;  // 启用追踪                     
  ITM->LAR = 0xC5ACCE55;  // 解锁 ITM                                                                                                                                                                                
  ITM->TER = 0x1;         // 启用端口 0                                            
  ITM->TPR = 0x0;         // 所有特权级都可访问                                                                                                                                                                      

  2. 包装 BASEPRI 操作函数                                                                                                                                                                                           
  #include <core_cm4.h>                                                                                                                                                                                              

  void trace_basepri_write(uint32_t value, const char* func, uint32_t line) {                                                                                                                                        
      // 发送函数名到 ITM                                                                                                                                                                                            
      const char* p = func;                                                                                                                                                                                          
      while (*p) {                                                                                                                                                                                                   
          ITM_SendChar(*p++);                                                                                                                                                                                        
      }                                                                            
      ITM_SendChar('|');                                                                                                                                                                                             

      // 发送行号                                                                                                                                                                                                    
      char buf[10];                                                                                                                                                                                                  
      snprintf(buf, sizeof(buf), "%lu", line);                                                                                                                                                                       
      p = buf;                                                                                                                                                                                                       
      while (*p) {                                                                 
          ITM_SendChar(*p++);                                                                                                                                                                                        
      }                                                                                                                                                                                                              
      ITM_SendChar('\n');                                                                                                                                                                                            

      // 实际写入 BASEPRI                                                                                                                                                                                            
      __set_BASEPRI(value);                                                                                                                                                                                          
  }                                                                                

  // 宏定义替换所有 BASEPRI 操作                                                                                                                                                                                     
  #define __set_BASEPRI traced(value) trace_basepri_write(value, __FUNCTION__, __LINE__)                                                                                                                             

  3. 在 CCS 中查看追踪                                                             
  - 打开 SWO Trace 窗口                                                                                                                                                                                              
  - 设置解码为 ASCII                                                               
  - 运行程序，你将看到类似输出：main|156 表示 main 函数第 156 行修改了 BASEPRI                                                                                                                                       

  方法2：使用数据断点（Watchpoint）                                                                                                                                                                                  

  步骤：                                                                                                                                                                                                             

  1. 获取 BASEPRI 寄存器地址                                                                                                                                                                                         
  // BASEPRI 是特殊寄存器，需要通过以下方式获取                                                                                                                                                                      
  uint32_t basepri_addr;                                                                                                                                                                                             
  __asm volatile("mrs %0, basepri" : "=r" (basepri_addr));                                                                                                                                                           

  2. 在 CCS 中设置数据断点                                                                                                                                                                                           
  - 打开 Breakpoints 视图                                                                                                                                                                                            
  - 右键 → New → New Breakpoint → Data Watchpoint                                                                                                                                                                    
  - 地址输入：0xE000E008（这是 BASEPRI 的地址）                                                                                                                                                                      
  - 访问类型：Write                                                                                                                                                                                                  
  - 条件：可选（如 value != 0）                                                                                                                                                                                      

  3. 自定义断点处理                                                                                                                                                                                                  
  // 在断点处自动记录调用栈                                                                                                                                                                                          
  void BASEPRI_Breakpoint_Handler(void) {                                                                                                                                                                            
      // 获取当前 PC 值                                                                                                                                                                                              
      uint32_t pc;                                                                                                                                                                                                   
      __asm volatile("mov %0, pc" : "=r" (pc));                                                                                                                                                                      

      // 记录到内存缓冲区                                                                                                                                                                                            
      static uint32_t call_trace[100];                                                                                                                                                                               
      static int trace_idx = 0;                                                                                                                                                                                      
      call_trace[trace_idx++] = pc;                                                                                                                                                                                  

      // 继续执行                                                                                                                                                                                                    
  }                                                                                                                                                                                                                  

  方法3：使用 MTB（Micro Trace Buffer）                                                                                                                                                                              

  步骤：                                                                                                                                                                                                             

  1. 启用 MTB 追踪                                                                                                                                                                                                   
  // 启用 MTB                  
  MTB->MASTER = MTB_MASTER_EN_Msk | MTB_MASTER_TSTART_Msk;                                                                                                                                                           
  MTB->POSITION = 0;                                                                                                                                                                                                 
  MTB->FLOW = 0;  // 循环缓冲模式        

  2. 包装 BASEPRI 操作                                                                                                                                                                                               
  void monitor_basepri_set(uint32_t value) {                                                                                                                                                                         
      // 在 MTB 中生成同步包                                                                                                                                                                                         
      MTB->PACKET = 0x80000000 | (__LINE__ & 0x7FFF);                                                                                                                                                                

      // 实际设置 BASEPRI                                                                                                                                                                                            
      __set_BASEPRI(value);                                                                                                                                                                                          
  }                                                                                

  3. 分析追踪数据                                                                                                                                                                                                    
  - 在 CCS 中导出 MTB 数据                                                                                                                                                                                           
  - 使用脚本解析程序流                                                                                                                                                                                               

  方法4：运行时仪器化（推荐）                                                                                                                                                                                        

  完整实现：                                                                                                                                                                                                         

  #include <stdint.h>                                                                                                                                                                                                
  #include <string.h>                                                                                                                                                                                                

  // 追踪缓冲区                                                                                                                                                                                                      
  typedef struct {                                                                                                                                                                                                   
      uint32_t pc;
      uint32_t basepri_old;                                                                                                                                                                                          
      uint32_t basepri_new;                                                        
      uint32_t timestamp;                                                                                                                                                                                            
  } basepri_trace_t;                                                                                                                                                                                                 

  #define TRACE_SIZE 64                                                                                                                                                                                              
  static basepri_trace_t trace_buffer[TRACE_SIZE];                                                                                                                                                                   
  static int trace_index = 0;                                                                                                                                                                                        

  // 获取调用者地址                                                                
  static inline uint32_t get_return_address(void) {                                                                                                                                                                  
      uint32_t addr;                                                                                                                                                                                                 
      __asm volatile("mov %0, lr" : "=r" (addr));                                                                                                                                                                    
      return addr;                                                                                                                                                                                                   
  }                                                                                                                                                                                                                  

  // 包装的 BASEPRI 设置函数                                                                                                                                                                                         
  void basepri_set_traced(uint32_t new_value) {                                                                                                                                                                      
      uint32_t old_value;                                                                                                                                                                                            
      __asm volatile("mrs %0, basepri" : "=r" (old_value));                                                                                                                                                          

      // 记录追踪信息                                                                                                                                                                                                
      trace_buffer[trace_index].pc = get_return_address();                                                                                                                                                           
      trace_buffer[trace_index].basepri_old = old_value;                                                                                                                                                             
      trace_buffer[trace_index].basepri_new = new_value;                                                                                                                                                             
      trace_buffer[trace_index].timestamp = DWT->CYCCNT;                                                                                                                                                             

      trace_index = (trace_index + 1) % TRACE_SIZE;                                                                                                                                                                  

      // 实际设置 BASEPRI                                                                                                                                                                                            
      __set_BASEPRI(new_value);                                                                                                                                                                                      
  }                                                                                                                                                                                                                  

  // 使用宏替换原有调用                                                            
  #define __set_BASEPRI(v) basepri_set_traced(v)                                                                                                                                                                     

  // 导出追踪数据              
  void dump_basepri_trace(void) {                                                                                                                                                                                    
      for (int i = 0; i < TRACE_SIZE; i++) {                                                                                                                                                                         
          int idx = (trace_index + i) % TRACE_SIZE;
          printf("[%d] PC: 0x%08X, BASEPRI: 0x%02X -> 0x%02X, Time: %u\\n",                                                                                                                                          
                 i, trace_buffer[idx].pc,                                                                                                                                                                            
                 trace_buffer[idx].basepri_old,                                                                                                                                                                      
                 trace_buffer[idx].basepri_new,                                                                                                                                                                      
                 trace_buffer[idx].timestamp);                                                                                                                                                                       
      }                                                                                                                                                                                                              
  }                                                                                                                                                                                                                  

  在 CCS 中的操作步骤                                                                                                                                                                                                

  1. 设置代码断点：                                                                                                                                                                                                  
    - 在 basepri_set_traced 函数入口处设置断点                                                                                                                                                                       
    - 或使用数据断点监控 BASEPRI 写入    
  2. 查看调用栈：                                                                                                                                                                                                    
    - 当断点命中时，查看 Call Stack 窗口                                           
    - 可以看到完整的调用链                                                                                                                                                                                           
  3. 使用追踪窗口：                                                                                                                                                                                                  
  View → SWO Trace → 配置为 ASCII 解码                                                                                                                                                                               
  4. 导出追踪数据：                                                                                                                                                                                                  
  Run → Trace → Export Trace Data                                                  

  高级技巧                                                                                                                                                                                                           

  1. 条件断点：只在 BASEPRI 设置为特定值时中断                                                                                                                                                                       
  2. 计数器：统计每个函数修改 BASEPRI 的次数                                       
  3. 时间戳：使用 DWT 周期计数器精确计时                                                                                                                                                                             

  推荐使用 方法4（运行时仪器化），因为它提供了最全面的信息，包括调用地址、旧值、新值和时间戳，而且开销相对较小。

⏺ 以上方法可以帮助你精确监控哪个函数在何时修改了 BASEPRI 寄存器。你可以根据项目需求选择最适合的方法，或者组合使用多种方法获得更全面的调试信息。