理一下时间线

[x86: fix pmd_bad and pud_bad to support huge pages](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?h=linux-3.10.y&id=cded932b75ab0a5f9181ee3da34a0a488d1a14fd)
这时候32位和64位的pmd_bad不同，32 位更接近现在的用法，但64更接近现在的pud_bad的用法。

32位


`

-#define	pmd_bad(x)	((pmd_val(x) & (~PAGE_MASK & ~_PAGE_USER)) != _KERNPG_TABLE)
+#define	pmd_bad(x)	((pmd_val(x) \
+			  & ~(PAGE_MASK | _PAGE_USER | _PAGE_PSE | _PAGE_NX)) \
+			 != _KERNPG_TABLE)
 
 
 `
 
 64位
 
 `
 
 static inline unsigned long pud_bad(pud_t pud)
 {
-	return pud_val(pud) & ~(PTE_MASK | _KERNPG_TABLE | _PAGE_USER);
+	return pud_val(pud) &
+		~(PTE_MASK | _KERNPG_TABLE | _PAGE_USER | _PAGE_PSE | _PAGE_NX);
 }
 
 static inline unsigned long pmd_bad(pmd_t pmd)
 {
-	return pmd_val(pmd) & ~(PTE_MASK | _KERNPG_TABLE | _PAGE_USER);
+	return pmd_val(pmd) &
+		~(PTE_MASK | _KERNPG_TABLE | _PAGE_USER | _PAGE_PSE | _PAGE_NX);
 }
 `
 
 
 这次commit之后，有地方错了。
 [Revert "x86: fix pmd_bad and pud_bad to support huge pages"](Revert "x86: fix pmd_bad and pud_bad to support huge pages")
 进行了还原
 
 修复了bug
 [fix PAE pmd_bad bootup warning](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?h=linux-3.10.y&id=aeed5fce37196e09b4dac3a1c00d8b7122e040ce)
 
 对64位中的pud_bad和pmd_bad进行了加强
 
 
 对32位和64位中的pud_bad进行了合并，但选择的是削弱的用法。
 https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?h=linux-3.10.y&id=a61bb29af47b0e4052566d25f3391894306a23fd
 
 最终通过找到了大佬们的讨论贴：
 https://groups.google.com/forum/#!msg/fa.linux.kernel/EDNikUkgab8/g1eyGlEdwa8J
 
 
