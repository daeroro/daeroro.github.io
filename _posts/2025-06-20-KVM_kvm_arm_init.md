---
title:  "KVM - kvm_arm_init"

categories:
  - KVM
tags:
  - [KVM, KVM_ARM]

toc: true
toc_sticky: true
 
date: 2025-06-20
last_modified_at: 2025-06-20
---
### (1) hypervisor mode 확인
```c
// arch/arm64/kvm/arm.c
2768 /* Initialize Hyp-mode and memory mappings on all CPUs */                       
2769 static __init int kvm_arm_init(void)                                            
2770 {                                                                               
2771         int err;                                                                
2772         bool in_hyp_mode;                                                       
2773                                                                                 
2774         if (!is_hyp_mode_available()) {                                         
2775                 kvm_info("HYP mode not available\n");                           
2776                 return -ENODEV;                                                 
2777         }                                                                       
2778                                                                                 
2779         if (kvm_get_mode() == KVM_MODE_NONE) {                                  
2780                 kvm_info("KVM disabled from command line\n");                   
2781                 return -ENODEV;                            
2782         } 
```
- 2774: is_hyp_mode_available() 함수에서 protected kvm이거나 EL2에서 booting 했는 지를 확인한다. 둘 다 아니면 false를 반환하여 KVM module init에 실패한다.
- 2779: kvm_get_mode()에서 kernel parameter 중에 하나인 kvm-arm.mode에 따라 kvm_mode에 설정된 값을 가져온다.
    - none: KVM_MODE_NONE
    - protected: KVM_MODE_PROTECTED
    - nvhe: KVM_MODE_DEFAULT
    - nested: KVM_MODE_NV

### (2) system register table 초기화
```c
// arch/arm64/kvm/arm.c
2784         err = kvm_sys_reg_table_init();                                         
2785         if (err) {                                                              
2786                 kvm_info("Error initializing system register tables");          
2787                 return err;                                                     
2788         }                                                                       
2789                                                                                 
2790         in_hyp_mode = is_kernel_in_hyp_mode();                                  
2791                                                                                 
2792         if (cpus_have_final_cap(ARM64_WORKAROUND_DEVICE_LOAD_ACQUIRE) ||        
2793             cpus_have_final_cap(ARM64_WORKAROUND_1508412))                      
2794                 kvm_info("Guests without required CPU erratum workarounds can deadlock system!\n" \
2795                          "Only trusted guests should be used on this system.\n");
2796          
```
- 2784: system register들에 대한 table이 유효한지 확인하고, reset하거나 sr_forward_xa에 store한다.
  - table 유효 확인 항목
    - sys_reg_descs
    - cp14_regs, cp14_64_regs: debug 관련 coprocessor register
    - cp15_regs, cp15_64_regs: system 관련(mmu enable, 등..) coprocessor register
    - invariant_sys_regs
    - sys_insn_descs
  - invariant_sys_regs는 reset 함수를 호출한다.
  - CGT(Coarse Grained Trap), FGT(Fine Grained Trap) 레지스터에 대한 trap_config를 sr_forward_xa에 store한다.
  - 또한, sys_reg_descs, sys_insn_descs 레지스터 trap_config도 sr_forward_xa에 store한다.
    - sr_forward_xa는 xarray 구조체로 레지스터 op-code를 key 값으로 trap_config의 object 주소를 관리한다.
