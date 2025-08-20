# 用户态中断

阅读 [Risc-V Extension N Implementation](https://gallium70.github.io/rv-n-ext-impl/ch2_2_user_trap_handle_flow.html) 记录。

用户态中断，简单来说由于使得中断异常处理可以绕过 OS 这一性能瓶颈，因此能够极大提高性能。

## 1 用户态中断异常处理流程

以下展示了用户态中断异常的处理流程，图中左侧的 `uie.UXIE` 根据此次中断的类型可能是 `uie` 寄存器中的 `USIE`, `UTIE`, `UEIE` 3 个字段之一，这些字段分别作为用户态软件中断、用户态时钟中断、用户态外部中断的使能位。详细细节参考[原文档：用户中断寄存器 (uip 与 uie)](https://gallium70.github.io/rv-n-ext-impl/ch2_1_n_ext_spec.html#%E7%94%A8%E6%88%B7%E4%B8%AD%E6%96%AD%E5%AF%84%E5%AD%98%E5%99%A8-uip-%E4%B8%8E-uie)。

![用户态中断异常处理流程](https://gallium70.github.io/rv-n-ext-impl/assets/user_trap_flow.drawio.svg)

## 2 




