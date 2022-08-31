---
title: "Struct padding"
date: 2022-08-31T02:37:29-07:00
description: "How offsets of struct members are determined."
tags: ["compilers", "x86_64"]
image: "img/padding.png"
---

Structs in C and C++ typically have padding inserted after members to ensure favorable alignment of struct members and
of the struct as a whole when it's in an array. On some architectures, such as SPARC, access to memory of a given size
must be aligned to a boundary of the same size---that is, if you're reading or writing four bytes at once, the address
you're accessing must be a multiple of four. If you try to access four bytes at an address that isn't a multiple of
four, architectures that require strict alignment will cause a fault that will crash the program. On some architectures,
like x86(_64) and ARM, misaligned accesses are permitted but occur more slowly and without any guarantee of atomicity.

To prevent misaligned access, compilers will arrange structs in memory such that each member is aligned properly, unless
you tell them not to by including `__attribute__((packed))`. To do this, it inserts empty bytes between members and at
the end as necessary. For example, this struct:

```c++
struct Foo {
	int8_t  a;
	int32_t b;
	int8_t  c;
	int16_t d;
	int64_t e;
	int8_t  f;
};
```

will have padding inserted to make it equivalent to this:

```c++
struct Padded {
	int8_t  a; // Offset: 0
	int8_t  pad1[3];
	int32_t b; // Offset: 4
	int8_t  c; // Offset: 8
	int8_t  pad2[1];
	int16_t d; // Offset: 10
	int8_t  pad3[4];
	int64_t e; // Offset: 16
	int8_t  f; // Offset: 24
	int8_t  pad4[7];
}; // Total size: 32 bytes
```

For integral types, the alignment requirement is the same as its width. If you have a struct within a struct, its
alignment requirement won't be the size of the inner struct but instead the highest alignment requirement among its
members. When I was looking into how to pad structs, that wasn't something that was explicitly mentioned in any of the
sources I found. It's something I found through trial and error when I wrote [a program](https://github.com/heimskr/llvmhack)
that uses LLVM's libraries to examine how it determines offsets for struct members.

Here's an algorithm in pseudo-C++ for finding the offset of a given struct's members (or you can look at the
[actual code](https://github.com/heimskr/ll2x/blob/master/src/compiler/PaddedStructs.cpp) used in my x86_64 compiler):

```c++
/** Increase `number` (if necessary) to be a multiple of `alignment`. */
template <typename T>
static T upalign(T number, T alignment) {
	return number + ((alignment - (number % alignment)) % alignment);
}

size_t memberOffset(const StructType &struct_type, size_t index) {
	size_t offset = 0;

	for (size_t i = 1; i <= index; ++i) {
		const size_t prev_width = struct_type.members[i - 1].width();
		const size_t alignment  = struct_type.members[i].alignment();
		offset = upalign(offset + prev_width, alignment);
	}

	return offset;
}
```

To find the total width of a struct after padding, take the offset of the last member, add the width of the last member
and upalign the result to the struct's alignment requirement.

Note that although this post was written with x86_64 in mind, everything should apply to other architectures as well.