# Paging

While the concept and usage of page table manipulation is nothing new, I thought I would share my implementation because I believe it is much easier to read and understand than other public information.

Firstly, I like to include the page table related structures. I stole these from Xerox.
```cpp
union VIRT_ADDR
{
	U64 value;
	struct
	{
		U64 offset_4kb : 12;
		U64 pt_index : 9;
		U64 pd_index : 9;
		U64 pdpt_index : 9;
		U64 pml4_index : 9;
		U64 reserved : 16;
	};
	struct
	{
		U64 offset_2mb : 21;
		U64 pd_index : 9;
		U64 pdpt_index : 9;
		U64 pml4_index : 9;
		U64 reserved : 16;
	};
	struct
	{
		U64 offset_1gb : 30;
		U64 pdpt_index : 9;
		U64 pml4_index : 9;
		U64 reserved : 16;
	};
	VIRT_ADDR(U64 x) : value(x)
	{
	}
};
union CR3
{
	U64 flags;
	struct
	{
		U64 reserved1 : 3;
		U64 page_level_write_through : 1;
		U64 page_level_cache_disable : 1;
		U64 reserved2 : 7;
		U64 pml4_pfn : 36;
		U64 reserved3 : 16;
	};
	CR3(U64 x) : flags(x)
	{

	}
};
union PML4E
{
	U64 value;
	struct
	{
		U64 present : 1;
		U64 writeable : 1;
		U64 user_supervisor : 1;
		U64 page_write_through : 1;
		U64 page_cache : 1;
		U64 accessed : 1;
		U64 ignore_1 : 1;
		U64 page_size : 1;
		U64 ignore_2 : 4;
		U64 pfn : 36;
		U64 reserved : 4;
		U64 ignore_3 : 11;
		U64 nx : 1;
	};
};
union PDPTE
{
	U64 value;
	struct
	{
		U64 present : 1;
		U64 rw : 1;
		U64 user_supervisor : 1;
		U64 page_write_through : 1;
		U64 page_cache : 1;
		U64 accessed : 1;
		U64 ignore_1 : 1;
		U64 large_page : 1;
		U64 ignore_2 : 4;
		U64 pfn : 36;
		U64 reserved : 4;
		U64 ignore_3 : 11;
		U64 nx : 1;
	};
};
union PDE
{
	U64 value;
	struct
	{
		U64 present : 1;
		U64 rw : 1;
		U64 user_supervisor : 1;
		U64 page_write_through : 1;
		U64 page_cache : 1;
		U64 accessed : 1;
		U64 ignore_1 : 1;
		U64 large_page : 1;
		U64 ignore_2 : 4;
		U64 pfn : 36;
		U64 reserved : 4;
		U64 ignore_3 : 11;
		U64 nx : 1;
	};
};
union PTE
{
	U64 value;
	struct
	{
		U64 present : 1;
		U64 rw : 1;
		U64 user_supervisor : 1;
		U64 page_write_through : 1;
		U64 page_cache : 1;
		U64 accessed : 1;
		U64 dirty : 1;
		U64 access_type : 1;
		U64 global : 1;
		U64 ignore_2 : 3;
		U64 pfn : 36;
		U64 reserved : 4;
		U64 ignore_3 : 7;
		U64 pk : 4;
		U64 nx : 1;
	};
};
```
Next, here is my translation code which collects all paging related information gathered from the translation process. This is more useful and modular than other implementations I have seen. Note that this does not handle large pages as it should.
```cpp
class TRANSLATION_RESULT
{
public:
	U64 pml4;
	PML4E pml4e;
	U64 pdpt;
	PDPTE pdpte;
	U64 pd;
	PDE pde;
	U64 pt;
	PTE pte;
	U64 pa;
};
TRANSLATION_RESULT Translate(U64 dtb, VIRT_ADDR va)
{
	U64 pml4 = dtb;
	PML4E pml4e;
	readPhys(pml4 + 8 * va.pml4_index, &pml4e, sizeof(PML4E));
	U64 pdpt = pml4e.pfn << 12;
	PDPTE pdpte;
	readPhys(pdpt + 8 * va.pdpt_index, &pdpte, sizeof(PDPTE));
	U64 pd = pdpte.pfn << 12;
	PDE pde;
	readPhys(pd + 8 * va.pd_index, &pde, sizeof(PDE));
	U64 pt = pde.pfn << 12;
	PTE pte;
	readPhys(pt + 8 * va.pt_index, &pte, sizeof(PTE));
	U64 physical_page = pte.pfn << 12;
	return { pml4, pml4e, pdpt, pdpte, pd, pde, pt, pte, physical_page + va.offset_4kb };
}
```
Now that we have translated the virtual address to the physical address and collected all translation related information, we can use it to change information about the paging structure.
Here is an example where I make a region of memory executable:
```cpp
for (int i = 0; i < BYTES_TO_PAGES(image.mod.size); i++)
{
	VIRT_ADDR va = alloc_base + i * PAGE_SIZE;
	auto translation = Translate(dtb, va);
	translation.pte.nx = false;
	writePhys(translation.pt + 8 * va.pt_index, &translation.pte, sizeof(PTE));
}
```
