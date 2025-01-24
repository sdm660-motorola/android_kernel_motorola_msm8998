# KernelSU object files
kernelsu-objs := ksu.o
kernelsu-objs += allowlist.o
kernelsu-objs += apk_sign.o
kernelsu-objs += sucompat.o
kernelsu-objs += throne_tracker.o
kernelsu-objs += core_hook.o
kernelsu-objs += ksud.o
kernelsu-objs += embed_ksud.o
kernelsu-objs += kernel_compat.o

kernelsu-objs += selinux/selinux.o
kernelsu-objs += selinux/sepolicy.o
kernelsu-objs += selinux/rules.o

# Compiler flags
ccflags-y += -I$(srctree)/security/selinux -I$(srctree)/security/selinux/include
ccflags-y += -I$(objtree)/security/selinux -include $(srctree)/include/uapi/asm-generic/errno.h

# Kernel module configuration
obj-$(CONFIG_KSU) += kernelsu.o

# .git check and versioning
ifeq ($(wildcard $(srctree)/$(src)/.git),)
    $(warning "KSU_GIT_VERSION not defined! It is better to make KernelSU-Next a git submodule!")
    ccflags-y += -DKSU_VERSION=16
else
    $(shell cd $(srctree)/$(src); \
        [ -f ../.git/shallow ] && git fetch --unshallow)
    KSU_GIT_VERSION := 1728
    $(eval KSU_VERSION=$(shell expr 10000 + $(KSU_GIT_VERSION) + 200))
    $(info -- KernelSU-Next version: $(KSU_VERSION))
    ccflags-y += -DKSU_VERSION=$(KSU_VERSION)
endif

# Compatibility checks
ifeq ($(shell grep -q " current_sid(void)" $(srctree)/security/selinux/include/objsec.h && echo 0 || echo 1),0)
    ccflags-y += -DKSU_COMPAT_HAS_CURRENT_SID
endif

ifeq ($(shell grep -q "struct selinux_state " $(srctree)/security/selinux/include/security.h && echo 0 || echo 1),0)
    ccflags-y += -DKSU_COMPAT_HAS_SELINUX_STATE
endif

# Manager package and expected values
ifndef KSU_NEXT_EXPECTED_SIZE
    KSU_NEXT_EXPECTED_SIZE := 0x3e6
endif

ifndef KSU_NEXT_EXPECTED_HASH
    KSU_NEXT_EXPECTED_HASH := 79e590113c4c4c0c222978e413a5faa801666957b1212a328e46c00c69821bf7
endif

ifdef KSU_MANAGER_PACKAGE
    ccflags-y += -DKSU_MANAGER_PACKAGE=\"$(KSU_MANAGER_PACKAGE)\"
    $(info -- KernelSU-Next Manager package name: $(KSU_MANAGER_PACKAGE))
endif

$(info -- KernelSU-Next Manager signature size: $(KSU_NEXT_EXPECTED_SIZE))
$(info -- KernelSU-Next Manager signature hash: $(KSU_NEXT_EXPECTED_HASH))

ccflags-y += -DEXPECTED_NEXT_SIZE=$(KSU_NEXT_EXPECTED_SIZE)
ccflags-y += -DEXPECTED_NEXT_HASH=\"$(KSU_NEXT_EXPECTED_HASH)\"

ccflags-y += -DKSU_COMPAT_GET_CRED_RCU
ccflags-y += -DKSU_UMOUNT

# Atomic function check
ifeq ($(shell grep -q "atomic_inc_not_zero" $(srctree)/kernel/cred.c && echo 0 || echo 1),0)
    ATOMIC_INC_FUNC = atomic_inc_not_zero
else ifeq ($(shell grep -q "atomic_long_inc_not_zero" $(srctree)/kernel/cred.c && echo 0 || echo 1),0)
    ATOMIC_INC_FUNC = atomic_long_inc_not_zero
else
    $(info -- KSU_NEXT: Neither atomic_inc_not_zero nor atomic_long_inc_not_zero found in kernel/cred.c)
endif

$(info -- KSU_NEXT: Using $(ATOMIC_INC_FUNC) in get_cred_rcu patch.)

# Add the get_cred_rcu function to cred.h if needed
ifneq ($(shell grep -q "^static inline const struct cred \*get_cred_rcu" $(srctree)/include/linux/cred.h && echo 0 || echo 1),0)
    GET_CRED_RCU = static inline const struct cred *get_cred_rcu(const struct cred *cred)\n\
{\n\
    struct cred *nonconst_cred = (struct cred *)cred;\n\
    if (!cred)\n\
        return NULL;\n\
    if (!$(ATOMIC_INC_FUNC)(&nonconst_cred->usage))\n\
        return NULL;\n\
    validate_creds(cred);\n\
    return cred;\n\
}\n
    $(shell echo -e "$(GET_CRED_RCU)" >> $(srctree)/include/linux/cred.h)
endif

# Suppress warnings
ccflags-y += -Wno-implicit-function-declaration -Wno-strict-prototypes -Wno-int-conversion -Wno-gcc-compat
ccflags-y += -Wno-declaration-after-statement -Wno-unused-function
