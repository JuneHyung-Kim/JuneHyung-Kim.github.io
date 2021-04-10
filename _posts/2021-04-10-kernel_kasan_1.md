---
title: "KASAN (1)"
date: 2021-04-10 
categories: linux
tags: memory compiler
---

# KASAN (Kernel Address Sanitizer)

`Address sanitizer`란

- _buffer overflow_나 _dangling pointer(use after free)_ 같은 `메모리 에러`를 감지하는 C/C++를 위한 빠른 memory error detector이다. 

- 이 툴은 구글의 오픈소스이다.

  > https://github.com/google/sanitizers/wiki/AddressSanitizer

- 컴파일러의 _instrumentation_ 기반이며 `shadow memory`와 directly-mapped 된다.

  - shadow memory와의 매핑은 `implementation details`에서 설명된다.

  > Instrumentation: 오류를 진단하거나 추적 정보를 쓰기 위해 제품의 성능 정도를 모니터하거나 측정하는 기능 

- 현재 LLVM Clang, GCC, Xcode 같은 컴파일러 안에 구현되어 있다.

---



Linux Kernel에서 user-space 어플리케이션에 대한 _address sanitizer_ 구현체가 이 `KASAN` 이다.

- GCC (v5.0 이후) 컴파일시 -fsanitize=kernel-address 플래그를 사용하거나 

  ```shell
  g++ -O -g -fsanitize=kernel-address hello_world.cc
  ```

- v4.0 이후의 Linux Kernel에서 Kconfig 옵션에 CONFIG_KASAN를 추가해 사용 할 수 있다.



## IMPLEMENTATION DETAILS

> 다른 블로그[^1]를 참고하여 작성되었으며 내용은 후에 보강되거나 다른 포스트에 추가 될 수 있음
>
> 참고한 내용은 x86_64 머신에서의 `KASAN_GENEGIC ` 구현 예시이다.
>
> Linux Kernel v5.0에는 Tag 기반의 KASAN이 추가되었다. (후에 포스팅 예정)

### `Shadow Memory` 

- aligned 8-byte의 메모리의 상태는 shared memory의 1-byte로 표현된다. 

- 결과적으로 커널 메모리의 1/8은 `shadow memory` 전용으로 사용된다.

- 다음 커널 함수는 각 메모리 접근마다 해당하는 `shadow memory` 주소 값을 반환한다.

  ```c
  static inline void *kasan_mem_to_shadow(const void *addr){
          return (void *)((unsigned long)addr >> KASAN_SHADOW_SCALE_SHIFT)
                  + KASAN_SHADOW_OFFSET;
  }
  ```

- 이때 찾은 섀도우 메모리에 저장된 `status`와 요청된 메모리 접근 크기를 기반으로 메모리 접근이 유효한지 확인한다.

|       Status        | Description                                                  |
| :-----------------: | :----------------------------------------------------------- |
|      **0x00**       | All 8 bytes are accessible.                                  |
| **0x00 < N < 0x08** | The lower `N` bytes are accessible. Bytes `8 - N` to byte `8` are not. |
|      **0xff**       | The page was freed.                                          |
|      **0xfe**       | A redzone for `kmalloc_large`.                               |
|      **0xfc**       | A redzone for a slub object.                                 |
|      **0xfb**       | The object was freed.                                        |
|      **0xf5**       | Stack use after return.                                      |
|      **0xf8**       | Stack use after scope.                                       |

- 아래와 같은 코드를 수행 할 때 `KASAN` 동작 예시

  ```c
  u8 *ptr = kmalloc(12, GFP_KERNEL); /* ptr=0xffff8882245bc8c0 */
  ...
  kfree(ptr);
  ```

![Screen Shot 2020-03-30 at 9.20.04 PM.png](https://images.squarespace-cdn.com/content/v1/5e1f51eb1bb1681137ea90b8/1585617606534-BO7BNURFEEF2DDE5GUON/ke17ZwdGBToddI8pDm48kP5BI7_FTd76le-lPItVC9lZw-zPPgdn4jUwVcJE1ZvWQUxwkmyExglNqGp0IvTJZUJFbgE-7XRK3dMEBRBhUpxS-3fpPb9Hzs_RNcbm7aR0da2F1LHOdIkLKfU0OmLzoENvpAQcdGgawAy950RpF3Q/Screen+Shot+2020-03-30+at+9.20.04+PM.png?format=750w)

> 그림 출처[^1]

### `Redzone`

- 각 `Shadow status`는 `redzone`이라는 영역에 둘러싸여 있다.
- 이는 연속 된 버퍼에서 overflow를 감지하기 위해 사용된다.

![Screen Shot 2020-03-30 at 9.21.43 PM.png](https://images.squarespace-cdn.com/content/v1/5e1f51eb1bb1681137ea90b8/1585617706119-0N8UDS9SFXRUEGG2Q1ZA/ke17ZwdGBToddI8pDm48kGB-D_MZ-H2UZAW3iR1XmXvlfiSMXz2YNBs8ylwAJx2qgRUppHe6ToX8uSOdETM-XldvY_sAIyUlfjhoEMtv77GjXhmoKzSnSRgz9XyEztnhYRmBLHakx0cslw3I_LnU_U3FnxsKBhnuRMQEU0TtF2k/Screen+Shot+2020-03-30+at+9.21.43+PM.png?format=750w)

[^1]: https://www.starlab.io/blog/kasan-what-is-it-how-does-it-work-and-what-are-the-strange-numbers-at-the-end



