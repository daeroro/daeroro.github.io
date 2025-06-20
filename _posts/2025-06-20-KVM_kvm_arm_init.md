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