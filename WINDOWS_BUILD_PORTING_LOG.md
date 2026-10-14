# WINDOWS BUILD PORTING LOG
## Bloodborne PC — Portabilidade Nativa para Windows 11

**Status do resultado:** Build concluído com sucesso. O jogo abriu no Windows 11 após todos os ajustes documentados abaixo.

---

## Índice

1. [Resumo do Resultado](#1-resumo-do-resultado)
2. [Identificação do Projeto e Repositório](#2-identificação-do-projeto-e-repositório)
3. [Ambiente de Compilação](#3-ambiente-de-compilação)
4. [Ferramentas Utilizadas](#4-ferramentas-utilizadas)
5. [Dependências e Categorização](#5-dependências-e-categorização)
6. [Estado do Repositório Original](#6-estado-do-repositório-original)
7. [Cronologia de Erros e Correções](#7-cronologia-de-erros-e-correções)
8. [Arquivos Modificados e Ajustes Locais](#8-arquivos-modificados-e-ajustes-locais)
9. [Reprodução do Build em Instalação Limpa](#9-reprodução-do-build-em-instalação-limpa)
10. [O que foi comprovado pelo resultado final](#10-o-que-foi-comprovado-pelo-resultado-final)
11. [Avisos e Problemas Pendentes](#11-avisos-e-problemas-pendentes)
12. [Melhorias Recomendadas para Reprodutibilidade](#12-melhorias-recomendadas-para-reprodutibilidade)
13. [Tabela de Status das Afirmações](#13-tabela-de-status-das-afirmações)

---

## 1. Resumo do Resultado

O projeto foi configurado e compilado com sucesso no Windows 11 usando MSYS2 UCRT64, GCC 16.1.0, CMake 4.3.3 e Ninja 1.13.2. O binário `out\bbport.exe` foi gerado e o jogo abriu. As principais intervenções foram:

- Substituição do mecanismo de descoberta de `miniz` (de `find_path`/`find_library` para `add_subdirectory` com o código-fonte upstream).
- Adição do submódulo `xbyak` e integração via `add_subdirectory` no CMake.
- Compilação manual inicial de `libatrac9.a` válida (o `build.bat` foi posteriormente corrigido para automatizar esse passo).
- Criação de um shim local `C:\msys64\ucrt64\include\vk_mem_alloc.h` para satisfazer a busca do CMake pelo header VMA fora do subdiretório `vma/`.
- Cópia manual dos headers de `magic_enum` para `C:\msys64\ucrt64\include\magic_enum\` (nenhum pacote MSYS2 os fornece).
- Desativação de LTO para o alvo `bbport` com `set_property(TARGET bbport PROPERTY INTERPROCEDURAL_OPTIMIZATION FALSE)`.

---

## 2. Identificação do Projeto e Repositório

| Campo              | Valor                                                          |
|--------------------|----------------------------------------------------------------|
| Nome do projeto    | bloodborne\_pc\_windows\_port                                  |
| Repositório remoto | https://github.com/Mrsuss60/bloodborne_pc_windows_port.git     |
| Branch atual       | `master`                                                       |
| Commit HEAD        | `e0a2a27` — "Remove leftover Linux scripts and packaging files" |
| Tag mais recente   | `v0.4.2`                                                       |
| Diretório local    | `E:\Native Windows Port of Bloodborne\bloodborne_pc_windows_port` |

> **Nota sobre comprimento de caminho:** o `build.bat` detecta automaticamente caminhos com mais de 70 caracteres e remonta o diretório como drive virtual `X:` usando `subst` para evitar erros de `MAX_PATH`. Essa verificação é executada antes de qualquer outra etapa do script.

---

## 3. Ambiente de Compilação

### 3.1 Sistema Operacional e Hardware

- **OS:** Windows 11
- **Shell utilizado:** PowerShell

### 3.2 Ferramentas de Build

| Ferramenta          | Versão       | Caminho                                  |
|---------------------|--------------|------------------------------------------|
| GCC / G++           | 16.1.0       | `C:\msys64\ucrt64\bin\gcc.exe`           |
| CMake               | 4.3.3        | `C:\msys64\ucrt64\bin\cmake.exe`         |
| Ninja               | 1.13.2       | `C:\msys64\ucrt64\bin\ninja.exe`         |
| ar                  | (GNU binutils)| `C:\msys64\ucrt64\bin\ar.exe`           |
| nm                  | (GNU binutils)| `C:\msys64\ucrt64\bin\nm.exe`           |
| pkg-config          | (MSYS2)      | `C:\msys64\ucrt64\bin\pkg-config.exe`    |
| glslang / glslangValidator | 16.2.0 | `C:\msys64\ucrt64\bin\glslang.exe` e `glslangValidator.exe` |

Todos os compiladores e ferramentas de build confirmados pelo CMakeCache em `out/gpu/CMakeCache.txt` (gerado durante a configuração real do projeto).

### 3.3 Variáveis de Ambiente Configuradas no PowerShell

As variáveis abaixo foram configuradas manualmente antes de executar `.\build.bat`:

```powershell
$env:Path = "C:\msys64\ucrt64\bin;C:\msys64\ucrt64\share\cmake\bin;C:\msys64\usr\bin;$env:Path"
$env:MINGW_DIR   = "C:\msys64\ucrt64"
$env:CMAKE_PREFIX_PATH = "C:\msys64\ucrt64;C:\msys64\ucrt64\share\cmake"
```

**Explicação de cada variável:**

- **`$env:Path`** — Acrescenta os diretórios `ucrt64\bin` e `usr\bin` do MSYS2 ao PATH do Windows. Isso é necessário porque o `build.bat` invoca `gcc`, `cmake`, `ninja`, `pkg-config`, `glslang`, `ar` e `git` como comandos soltos. Sem o PATH correto, o shell do Windows não encontra nenhum desses executáveis.
  - `C:\msys64\ucrt64\bin` — compiladores, linker, ar, nm, cmake, ninja, glslang.
  - `C:\msys64\ucrt64\share\cmake\bin` — modules e scripts CMake auxiliares.
  - `C:\msys64\usr\bin` — utilitários Unix (bash, pacman, etc.) usados por scripts auxiliares.

- **`$env:MINGW_DIR`** — Consumida pelo `build.bat` (`set "PATH=%MINGW_DIR%\bin;%PATH%"` dentro do script). Aponta explicitamente para o ambiente UCRT64. Sem essa variável, o script tenta detectar `C:\msys64\mingw64\bin` como fallback.

- **`$env:CMAKE_PREFIX_PATH`** — Acumulada pelo `build.bat` junto com outros caminhos e passada ao CMake via `-DCMAKE_PREFIX_PATH=`. Permite que `find_package()` e `pkg_check_modules()` localizem bibliotecas instaladas no UCRT64.

### 3.4 Como o `build.bat` Configura o Ambiente Internamente

O script `build.bat` realiza as seguintes operações em sequência:

1. Detecta e remonta o projeto como drive `X:` se o caminho tiver mais de 70 caracteres.
2. Cria `out\` e `out\gpu\` se não existirem.
3. Acrescenta `%MINGW_DIR%\bin` ao PATH interno (ou `C:\msys64\mingw64\bin` como fallback, se `MINGW_DIR` não estiver definido).
4. Acrescenta `C:\Program Files\CMake\bin` ao PATH, se existir.
5. Configura `git config --local core.longpaths true` para evitar falhas com headers do FidelityFX.
6. Inicializa submódulos automaticamente se `gpu\third_party\fsr-vulkan\CMakeLists.txt` não existir.
7. **Compila a LibAtrac9** se `out\libatrac9.a` não existir:
   - Compila todos os `.c` de `third_party\LibAtrac9\C\src\` com `gcc -std=c99 -O2 -g -w`.
   - Empacota os `.o` em `out\libatrac9.a` com `ar rcs`.
8. Aplica `patches\fsr_vulkan_mingw.patch` ao submódulo FSR-Vulkan se o patch ainda não foi aplicado.
9. Configura o CMake com Ninja (`-S gpu -B out/gpu -G Ninja`) se `out\gpu\build.ninja` não existir.
   - Monta `CMAKE_PREFIX_PATH` incluindo `SDL3_DIR`, `VULKAN_SDK`, `C:/msys64/mingw64` (hardcoded) e `CMAKE_PREFIX_PATH` do ambiente.
10. Executa o build: `cmake --build out/gpu --target bbgpu bbport bb-gpu-capabilities -- -j 1`.
11. Copia `out\bbport.exe` para `out\bb-probe.exe`.
12. Executa `python scripts\stage_dlls.py` para posicionar DLLs de runtime.

**Ponto crítico:** o script hardcoda `C:\msys64\mingw64` no `CMAKE_PREFIX_PATH` interno, independentemente do valor de `MINGW_DIR`. Isso significa que, mesmo num ambiente UCRT64, o `CMAKE_PREFIX_PATH` enviado ao CMake inclui `C:/msys64/mingw64`. Na configuração desta máquina, o diretório `C:\msys64\mingw64` existe mas não é o ambiente ativo. O CMake usa como compilador os binários encontrados primeiro no PATH, que foram os do UCRT64 — confirmado pelo CMakeCache. O fato de `C:/msys64/mingw64` aparecer no `CMAKE_PREFIX_PATH` não causou erros porque o CMake procura os prefixos em ordem e encontrou as bibliotecas no UCRT64 antes.

---

## 4. Ferramentas Utilizadas

| Ferramenta       | Uso identificado                                          |
|------------------|-----------------------------------------------------------|
| PowerShell       | Execução do `build.bat`, configuração de variáveis de ambiente |
| Git              | Controle de versão, inicialização de submódulos, aplicação de patches |
| GCC 16.1.0       | Compilação de C/C++, compilação manual da LibAtrac9       |
| `ar`             | Criação de `libatrac9.a` a partir dos `.o`                |
| `nm`             | Verificação de símbolos em `libatrac9.a` e `libatrac9_test.a` |
| CMake 4.3.3      | Configuração do build system                              |
| Ninja 1.13.2     | Execução do build                                         |
| `pkg-config`     | Descoberta de FFmpeg, xxhash, SDL3 (fallback)             |
| `glslang`/`glslangValidator` | Compilação dos shaders GLSL para headers C     |
| Editor de texto  | Modificação de `gpu/CMakeLists.txt`                       |
| Python           | `scripts/stage_dlls.py` (staging de DLLs de runtime)     |

---

## 5. Dependências e Categorização

### 5.1 Submódulos Git (registrados em `.gitmodules` atual)

| Componente           | URL                                          | Diretório                       | Commit                |
|----------------------|----------------------------------------------|---------------------------------|-----------------------|
| FSR-Vulkan           | https://github.com/FireBurn/FSR-Vulkan.git   | `gpu/third_party/fsr-vulkan`    | `c64f0934` (heads/main) |
| Dear ImGui           | https://github.com/ocornut/imgui.git         | `gpu/third_party/imgui`         | `f1cc2ae1` (v1.62+)   |
| LibAtrac9            | https://github.com/Thealexbarney/LibAtrac9.git | `third_party/LibAtrac9`       | `efca2e3a` (v0.0.3+)  |
| Xbyak                | https://github.com/herumi/xbyak.git          | `gpu/third_party/xbyak`         | `cd6bad3c` (v7.43)    |

> **Confirmado:** Xbyak foi adicionado como novo submódulo durante o porting. Os três primeiros (FSR-Vulkan, ImGui, LibAtrac9) existiam no repositório original.

### 5.2 Código-Fonte Incluído Diretamente (não submódulo, não pacote)

| Componente   | Diretório                        | Origem                                      | Status Git       |
|--------------|----------------------------------|---------------------------------------------|------------------|
| miniz        | `gpu/third_party/miniz/`         | https://github.com/richgel999/miniz (clone) | Não rastreado (arquivo independente, não submódulo) |
| sirit        | `gpu/third_party/sirit/`         | Incluído no repositório                     | Rastreado        |
| Xbyak headers| `gpu/third_party/xbyak/`         | Submódulo herumi/xbyak                      | Rastreado (submódulo adicionado) |
| gcn headers  | `gpu/third_party/gcn/`           | Incluído no repositório                     | Rastreado        |
| half         | `gpu/third_party/half/`          | Incluído no repositório                     | Rastreado        |
| fonts        | `gpu/third_party/fonts/`         | Incluído no repositório                     | Rastreado        |

> **Importante — miniz:** O diretório `gpu/third_party/miniz/` contém um repositório Git independente (tem `.git` próprio), mas **não está registrado como submódulo** do projeto principal. Isso significa que `git submodule update --init` não o baixará. É necessário clonar manualmente ou registrá-lo como submódulo.

### 5.3 Pacotes MSYS2 UCRT64 Instalados via `pacman`

Confirmados pela saída de `pacman -Q` na máquina:

| Pacote MSYS2 (ucrt64)                            | Versão      | Componente     |
|--------------------------------------------------|-------------|----------------|
| `mingw-w64-ucrt-x86_64-vulkan-headers`           | 1~1.4.350.0 | Vulkan Headers |
| `mingw-w64-ucrt-x86_64-vulkan-loader`            | 1~1.4.350.0 | Vulkan Loader  |
| `mingw-w64-ucrt-x86_64-vulkan-memory-allocator`  | 3.4.0       | VMA            |
| `mingw-w64-ucrt-x86_64-fmt`                      | 12.1.0      | fmt            |
| `mingw-w64-ucrt-x86_64-boost`                    | 1.91.0      | Boost          |
| `mingw-w64-ucrt-x86_64-xxhash`                   | 0.8.3       | xxHash         |
| `mingw-w64-ucrt-x86_64-ffmpeg`                   | 8.1.1       | FFmpeg         |
| `mingw-w64-ucrt-x86_64-robin-map`                | 1.4.1       | tsl-robin-map  |
| `mingw-w64-ucrt-x86_64-glslang`                  | 16.2.0      | glslang        |
| `mingw-w64-ucrt-x86_64-zydis`                    | 4.1.1       | Zydis          |
| `mingw-w64-ucrt-x86_64-sdl3`                     | 3.4.10      | SDL3           |
| `mingw-w64-ucrt-x86_64-zlib`                     | 1.3.2       | zlib           |

> **`magic_enum`:** Não existe pacote MSYS2 com esse nome. Os headers foram instalados manualmente — ver Seção 8.

### 5.4 Bibliotecas Compiladas durante o Build

| Biblioteca       | Como é gerada                                  | Localização final        |
|------------------|------------------------------------------------|--------------------------|
| `libatrac9.a`    | GCC + `ar`, a partir de `third_party/LibAtrac9/C/src/*.c` | `out/libatrac9.a` |
| `miniz` (target) | `add_subdirectory` compila junto com bbgpu     | `out/gpu/third_party/miniz/` |
| `xbyak` (target) | `add_subdirectory`, header-only em essência    | `out/gpu/third_party/xbyak/` |
| `sirit`          | `add_subdirectory` compila junto com bbgpu     | `out/gpu/third_party/sirit/` |
| `bbgpu.a`        | Compilação principal — video core + shaders    | `out/gpu/libbbgpu.a`     |
| `bbport.exe`     | Executável final linking bbgpu + LibAtrac9     | `out/bbport.exe`         |
| `bb-gpu-capabilities.exe` | Ferramenta de diagnóstico Vulkan      | `out/`                   |

### 5.5 Como o CMake Descobriu Cada Dependência (confirmado pelo CMakeCache)

| Dependência       | Mecanismo                         | Caminho resolvido                                |
|-------------------|-----------------------------------|--------------------------------------------------|
| Vulkan            | `find_package(Vulkan REQUIRED)`   | `C:/msys64/ucrt64` (via CMAKE_PREFIX_PATH)       |
| fmt               | `find_package(fmt REQUIRED)`      | `C:/msys64/ucrt64/lib/cmake/fmt`                 |
| Boost             | `find_package(Boost REQUIRED)`    | `C:/msys64/ucrt64/lib/cmake/Boost-1.91.0`        |
| SDL3              | `find_package(SDL3 CONFIG)`       | `C:/msys64/ucrt64/lib/cmake/SDL3`                |
| tsl-robin-map     | `find_package(tsl-robin-map)`     | `C:/msys64/ucrt64`                               |
| VMA               | `find_path(VMA_INC vk_mem_alloc.h)` | `C:/msys64/ucrt64/include` (pelo shim)         |
| magic_enum        | `find_path(MAGIC_ENUM_INC ...)`   | `C:/msys64/ucrt64/include` (headers copiados)    |
| xxHash            | `pkg_check_modules(libxxhash)`    | `C:/msys64/ucrt64`                               |
| FFmpeg            | `pkg_check_modules(libavformat…)` | `C:/msys64/ucrt64`                               |
| Zydis             | `find_package(Zydis REQUIRED)`    | `C:/msys64/ucrt64/lib/cmake/zydis`               |
| glslang           | `find_program(GLSLANG_COMPILER…)` | `C:/msys64/ucrt64/bin/glslang.exe`               |
| miniz             | `add_subdirectory(third_party/miniz)` | `gpu/third_party/miniz/` (fonte local)       |
| Xbyak             | `add_subdirectory(third_party/xbyak)` | `gpu/third_party/xbyak/` (submódulo)        |

---

## 6. Estado do Repositório Original

### 6.1 Referência Utilizada para Comparação

O commit `e0a2a27` (HEAD do remote `origin/master`) é usado como ponto de referência. O diff é obtido com `git diff HEAD`. As alterações locais ainda **não foram commitadas**.

### 6.2 Submódulos no `.gitmodules` Original (antes das alterações locais)

```
[submodule "gpu/third_party/fsr-vulkan"]   → FireBurn/FSR-Vulkan
[submodule "gpu/third_party/imgui"]        → ocornut/imgui
[submodule "third_party/LibAtrac9"]        → Thealexbarney/LibAtrac9
```

Xbyak **não estava** registrado como submódulo.

### 6.3 Comportamento Original do CMake para `miniz` (WIN32)

O `gpu/CMakeLists.txt` original tentava localizar `miniz` instalado no sistema via:
```cmake
if (WIN32)
    if (NOT TARGET miniz::miniz)
        add_library(miniz::miniz INTERFACE IMPORTED)
        find_path(MINIZ_INC NAMES miniz.h PATHS "$ENV{MINGW_PREFIX}/include" "C:/msys64/mingw64/include")
        find_library(MINIZ_LIB NAMES miniz libminiz.a PATHS "$ENV{MINGW_PREFIX}/lib" "C:/msys64/mingw64/lib")
    endif()
else()
    find_package(miniz REQUIRED)
endif()
```

O CMakeCache registra `MINIZ_INC:PATH=MINIZ_INC-NOTFOUND` e `MINIZ_LIB:FILEPATH=MINIZ_LIB-NOTFOUND`, confirmando que a abordagem original **falhou** nesta máquina (nenhum pacote MSYS2 instala `miniz` em `include/miniz.h`).

### 6.4 Xbyak no CMake Original

O `gpu/CMakeLists.txt` original **não continha nenhuma referência a Xbyak** — nem `find_package`, nem `add_subdirectory`, nem `target_link_libraries`. O header `xbyak/xbyak.h` é utilizado pelo código-fonte do shadPS4 embutido no projeto, portanto a ausência causaria erro de compilação.

### 6.5 Limitações da Comparação

- Não existe cópia do repositório no estado anterior aos ajustes locais (sem um clone limpo separado). A comparação se baseia inteiramente no `git diff HEAD`.
- O arquivo `build-log.txt` capturou apenas um erro de tentativa de build com PATH incorreto (sem `gcc` no PATH), não os erros anteriores relacionados a miniz, xbyak e LTO.
- Os erros das etapas anteriores (VMA, magic_enum, LibAtrac9) foram relatados pelo usuário; não há log CMake ou de linker preservado para eles.

---

## 7. Cronologia de Erros e Correções

### Etapa A — Preparação do Ambiente Windows

**Contexto:** Um projeto originalmente desenvolvido para Linux/PS4 (via shadPS4) sendo portado para compilação nativa no Windows requer atenção especial a:
- Localização de bibliotecas: o Linux usa `pkg-config` com prefixos padronizados; o Windows com MSYS2 precisa de `CMAKE_PREFIX_PATH` explícito.
- Convenções de chamada: funções com `__attribute__((sysv_abi))` usam a ABI de chamada do System V AMD64, diferente da convenção padrão do Windows. Em `src/runtime.h`, a macro `ABI` é definida como `__attribute__((sysv_abi))`, e as funções `restore_guest_fs`, `pre_import_hook`, `post_import_hook` são declaradas com ela.
- Opções de linker: o executável `bbport` usa `-Wl,--disable-dynamicbase -Wl,--disable-high-entropy-va` para compatibilidade com endereçamento fixo.
- Bibliotecas Windows adicionais: `ws2_32`, `dbghelp`, `ntdll`, `winmm` são linkadas explicitamente no alvo `bbport` no WIN32.

**Configuração utilizada:** As variáveis de ambiente descritas na Seção 3.3 foram configuradas no PowerShell antes de executar `.\build.bat`. O `build.bat` consome `MINGW_DIR` para estender o PATH interno.

---

### Etapa B — VMA: Header em Subdiretório Inesperado

**Erro:** O CMake não encontrava `vk_mem_alloc.h`. A busca original era:
```cmake
find_path(VMA_INC NAMES vk_mem_alloc.h PATHS "$ENV{MINGW_PREFIX}/include" "C:/msys64/mingw64/include")
```
O pacote MSYS2 `mingw-w64-ucrt-x86_64-vulkan-memory-allocator` (v3.4.0) instala o header em:
```
C:\msys64\ucrt64\include\vma\vk_mem_alloc.h
```
Porém `find_path` busca pelo nome `vk_mem_alloc.h` diretamente no diretório de includes, não dentro de `vma/`. O arquivo não foi encontrado na busca padrão (`VMA_INC:PATH=MINIZ_INC-NOTFOUND` no CMakeCache seria o equivalente para VMA).

**Correção local aplicada:** Foi criado manualmente o arquivo:
```
C:\msys64\ucrt64\include\vk_mem_alloc.h
```
Com o conteúdo:
```cpp
#include <vma/vk_mem_alloc.h
```
> **Nota:** O arquivo está sem o `>` de fechamento — `#include <vma/vk_mem_alloc.h` (sem fechar). Esse formato pode funcionar em compiladores tolerantes mas é tecnicamente malformado. O correto seria `#include <vma/vk_mem_alloc.h>`. **Pendente de verificação se isso causa problemas de compilação em algum caso.**

O `find_path` então encontra `vk_mem_alloc.h` em `C:\msys64\ucrt64\include` e resolve `VMA_INC` para esse diretório. O target `GPUOpen::VulkanMemoryAllocator` recebe esse diretório como include e o código que faz `#include "vk_mem_alloc.h"` encontra o shim, que por sua vez inclui o arquivo real do pacote.

**Confirmado no CMakeCache:**
```
VMA_INC:PATH=C:/msys64/ucrt64/include
```

**Característica:** Esta é uma **alteração local não versionada**. O arquivo existe em `C:\msys64\ucrt64\include\` mas não está dentro do repositório e não é rastreado pelo Git.

---

### Etapa C — magic_enum: Sem Pacote MSYS2

**Problema:** A macro CMake original tentava localizar `magic_enum/magic_enum.hpp` via:
```cmake
find_path(MAGIC_ENUM_INC NAMES magic_enum/magic_enum.hpp PATHS "$ENV{MINGW_PREFIX}/include" "C:/msys64/mingw64/include")
```
Não existe pacote `mingw-w64-ucrt-x86_64-magic_enum` no repositório padrão do MSYS2 (confirmado por `pacman -Q`). A consulta `pacman -Qo` confirma que os headers instalados em `C:\msys64\ucrt64\include\magic_enum\` não pertencem a nenhum pacote rastreado pelo pacman.

**Correção local aplicada:** Os headers foram copiados manualmente para:
```
C:\msys64\ucrt64\include\magic_enum\
```
Os arquivos presentes (todos com data 13/10/2026):
- `magic_enum.hpp` (82.392 bytes)
- `magic_enum_all.hpp`
- `magic_enum_containers.hpp`
- `magic_enum_flags.hpp`
- `magic_enum_format.hpp`
- `magic_enum_fuse.hpp`
- `magic_enum_iostream.hpp`
- `magic_enum_switch.hpp`
- `magic_enum_utility.hpp`

O `find_path` busca `magic_enum/magic_enum.hpp` e encontra o arquivo em `C:\msys64\ucrt64\include\magic_enum\magic_enum.hpp`, resolvendo `MAGIC_ENUM_INC` para `C:\msys64\ucrt64\include`.

**Confirmado no CMakeCache:**
```
MAGIC_ENUM_INC:PATH=C:/msys64/ucrt64/include
```

**Característica:** **Alteração local não versionada.** Os headers existem apenas na máquina. Necessário documentar a fonte dos headers para reprodutibilidade.

> **Inferência:** Os headers provavelmente são da release oficial em https://github.com/Neargye/magic_enum (header-only, MIT). A versão exata não é verificável sem o conteúdo dos arquivos.

---

### Etapa D — miniz: Substituição de `find_path`/`find_library` por `add_subdirectory`

**Problema original:** O CMake tentava encontrar `miniz` instalado no sistema. No UCRT64, `miniz` não está disponível como pacote MSYS2. O CMakeCache registra:
```
MINIZ_INC:PATH=MINIZ_INC-NOTFOUND
MINIZ_LIB:FILEPATH=MINIZ_LIB-NOTFOUND
```
Com isso, o target `miniz::miniz` era criado sem includes e sem biblioteca, e a linkagem falharia.

**Solução aplicada:** O código-fonte do miniz upstream foi obtido em `gpu/third_party/miniz/` (repositório independente com `.git` próprio, confirmado pela estrutura do diretório). O `gpu/CMakeLists.txt` foi alterado para compilar o miniz junto com o projeto:

```cmake
# Antes do add_subdirectory: desativa artefatos desnecessários do miniz
set(BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
set(BUILD_TESTS    OFF CACHE BOOL "" FORCE)
set(INSTALL_PROJECT OFF CACHE BOOL "" FORCE)

if (NOT TARGET miniz)
    add_subdirectory(third_party/miniz EXCLUDE_FROM_ALL)
endif()

if (NOT TARGET miniz::miniz)
    add_library(miniz::miniz ALIAS miniz)
endif()
```

**O que cada parte faz:**
- `BUILD_EXAMPLES/TESTS/INSTALL OFF` — impede que o CMakeLists do miniz tente compilar exemplos e testes ou registrar regras de instalação, que causariam dependências desnecessárias.
- `add_subdirectory(third_party/miniz EXCLUDE_FROM_ALL)` — inclui o CMakeLists do miniz no build, mas com `EXCLUDE_FROM_ALL` para que ele não faça parte do alvo `all`. O target `miniz` é criado pelo CMakeLists do upstream (confirma-se no CMakeLists do miniz: `project(miniz C)` → target `miniz`).
- `add_library(miniz::miniz ALIAS miniz)` — cria o alias esperado pelo restante do código CMake (`target_link_libraries(... miniz::miniz ...)`), mapeando o nome sem namespace para o nome com namespace.

O `find_package(Zydis REQUIRED)` permanece inalterado após esse bloco, confirmando que a integração do miniz não afetou a descoberta de Zydis.

**Status do `gpu/third_party/miniz` no Git:**
```
?? gpu/third_party/miniz/
```
É um diretório não rastreado — contém `.git` próprio mas não está registrado como submódulo do projeto principal. Para reproduzir, é preciso clonar manualmente ou registrar como submódulo.

---

### Etapa E — Xbyak: Header Ausente, Submódulo Adicionado

**Erro original:** O código-fonte do shadPS4 embutido no projeto usa `#include <xbyak/xbyak.h>` para geração dinâmica de código x86-64. O repositório original não continha o Xbyak, e o `gpu/CMakeLists.txt` original não tinha nenhuma referência a ele. A compilação falharia com `xbyak/xbyak.h: No such file or directory`.

**Solução aplicada:**

1. O repositório https://github.com/herumi/xbyak.git foi adicionado como submódulo em `gpu/third_party/xbyak/`. Commit atual: `cd6bad3c` (tag v7.43).

2. Em `.gitmodules`, foi adicionado:
   ```
   [submodule "gpu/third_party/xbyak"]
       path = gpu/third_party/xbyak
       url = https://github.com/herumi/xbyak.git
   ```

3. Em `gpu/CMakeLists.txt`, foi adicionado o bloco:
   ```cmake
   if (NOT TARGET xbyak::xbyak)
       add_subdirectory(third_party/xbyak EXCLUDE_FROM_ALL)
   endif()
   ```

4. Em `target_link_libraries` do alvo `bbgpu` para WIN32, `xbyak::xbyak` foi adicionado:
   ```cmake
   target_link_libraries(bbgpu PUBLIC ... xbyak::xbyak ...)
   ```

**Verificado:** O diretório `gpu/third_party/xbyak` existe, contém `.git` próprio e aponta para `origin https://github.com/herumi/xbyak.git`. O `git submodule status` mostra o submódulo com commit `cd6bad3c` (v7.43). O CMakeCache confirma:
```
xbyak_SOURCE_DIR: .../gpu/third_party/xbyak
xbyak_BINARY_DIR: .../out/gpu/third_party/xbyak
```

**Confirmado:** A adição do submódulo e a integração via CMake estão em vigor e foram efetivas.

---

### Etapa F — LibAtrac9: Biblioteca Estática Inválida ou Inexistente

**Problema:** A LibAtrac9 é usada pelo projeto para decodificação de áudio ATRAC9. O binário `bbport` é linkado com `"${CMAKE_CURRENT_SOURCE_DIR}/../out/libatrac9.a"`. Quando `out/libatrac9.a` não existe ou tem apenas 8 bytes (arquivo vazio ou truncado), o linker falha com símbolos ausentes:

- `Atrac9ReleaseHandle`
- `Atrac9GetHandle`
- `Atrac9InitDecoder`
- `Atrac9GetCodecInfo`
- `Atrac9Decode`

**Comportamento do `build.bat` original:** O script condiciona a compilação da LibAtrac9 à inexistência do arquivo:
```bat
if not exist "out\libatrac9.a" (
    ...compilar...
)
```
Um arquivo existente — mesmo com 8 bytes, inválido — impedia a recompilação. O script **não valida** se a biblioteca existente contém os símbolos esperados.

**Log de build capturado (`build-log.txt`):** O único log disponível registra um erro diferente — tentativa de build sem `gcc` no PATH:
```
Building LibAtrac9
'gcc' não é reconhecido como um comando interno
Failed to compile LibAtrac9 source: third_party\LibAtrac9\C\src\band_extension.c
```
Isso confirma que uma tentativa de build foi feita **antes** de configurar o PATH corretamente.

**Correção aplicada manualmente:**
1. Configuração do PATH para incluir `C:\msys64\ucrt64\bin`.
2. Criação do diretório `out\atrac9_test\`.
3. Compilação individual de cada `.c`:
   ```powershell
   # Equivalente ao que o build.bat faz, mas executado manualmente com PATH correto
   # Para cada arquivo em third_party\LibAtrac9\C\src\*.c:
   gcc -std=c99 -O2 -g -w -c <arquivo.c> -o out\atrac9_test\<arquivo>.o
   ```
4. Empacotamento:
   ```powershell
   ar rcs out\libatrac9_test.a out\atrac9_test\*.o
   ```
5. Verificação com `nm`:
   ```
   libatrac9.o: T Atrac9Decode, T Atrac9GetCodecInfo, T Atrac9GetHandle,
                T Atrac9InitDecoder, T Atrac9ReleaseHandle
   ```
6. Cópia para `out\libatrac9.a`.

**Estado atual verificado:**
- `out/libatrac9.a` — 211.996 bytes, contém todos os 5 símbolos esperados. **Confirmado por `nm`.**
- `out/libatrac9_test.a` — 211.996 bytes idêntico (arquivo intermediário preservado).
- `out/atrac9/` — contém os 13 arquivos `.o` de compilação da LibAtrac9.

O `build.bat` atual já contém a lógica de compilação automática da LibAtrac9. Após configurar o PATH corretamente, a geração é automática quando `out\libatrac9.a` não existe. O problema original ocorreu por falta de `gcc` no PATH.

**Recomendação futura (não implementada nesta tarefa):** O script deveria validar que `out\libatrac9.a` contém ao menos um símbolo `Atrac9*` após criá-la, e também deveria detectar e rejeitar uma biblioteca existente de tamanho zero ou muito pequeno (< 1.000 bytes como heurística).

---

### Etapa G — Falha de LTO no Linker

**Erro:** Após resolver todas as dependências e chegar à etapa de linkagem final do `bbport`, o linker (via LTO) falhou com referências a símbolos em seções descartadas:

- `pre_import_hook`
- `post_import_hook`
- `restore_guest_fs`
- `runtime_thread_get_tcb`

As mensagens típicas nesse caso indicam que o plugin LTO/linker descartou as seções que contêm definições dessas funções, enquanto o assembly embutido em `src/probe.c` as referencia diretamente por nome.

**Causa técnica (inferência):** Em `src/probe.c`, as funções `pre_import_hook`, `post_import_hook` e `restore_guest_fs` são declaradas com `ABI` (`__attribute__((sysv_abi))`), e o código assembly inline as chama via:
```c
"   call pre_import_hook\n"
"   call post_import_hook\n"
"   call restore_guest_fs\n"
"   call runtime_thread_get_tcb\n"
```
O LTO do GCC pode, durante a otimização interprocedural, marcar funções como internas ou descartáveis se não detectar referências externas suficientes. Referências em strings de assembly inline (`asm volatile(...)`) podem não ser tratadas como referências simbólicas reais pelo analisador LTO, levando ao descarte das definições.

> **Nota:** Esta é uma **inferência técnica** baseada no padrão de erro. Não existe log do linker preservado que demonstre a causa interna definitiva. O comportamento é consistente com erros documentados de LTO + inline assembly no GCC em Windows.

**A configuração LTO do projeto:**
```cmake
option(BB_LTO "Link-time optimization of bbgpu" ON)
if (BB_LTO)
    include(CheckIPOSupported)
    check_ipo_supported(RESULT bb_ipo_ok OUTPUT bb_ipo_error)
    if (bb_ipo_ok)
        set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)  # habilita LTO globalmente
    endif()
endif()
```
`CMAKE_INTERPROCEDURAL_OPTIMIZATION ON` ativa LTO para **todos** os alvos no escopo, incluindo `bbport`.

**Correção aplicada:** Imediatamente após a declaração de `add_executable(bbport ...)`, foi adicionado:
```cmake
set_property(TARGET bbport PROPERTY INTERPROCEDURAL_OPTIMIZATION FALSE)
```
Isso desativa a propriedade de otimização interprocedural especificamente para `bbport`, sem remover o LTO da biblioteca `bbgpu` (que não tem as referências de assembly problemáticas).

**Confirmado no `gpu/CMakeLists.txt` atual:**
```cmake
add_executable(bbport
    ../src/probe.c
    ${RUNTIME_SOURCES}
    ../src/vulkan_smoke.c
)
set_property(TARGET bbport PROPERTY INTERPROCEDURAL_OPTIMIZATION FALSE)
```

**Resultado:** O build completou e o jogo abriu após essa mudança.

**Limitação conhecida:** A propriedade `INTERPROCEDURAL_OPTIMIZATION FALSE` controla o comportamento CMake para geração de flags. Se flags `-flto` forem passadas explicitamente por outras vias (variáveis de ambiente, caches CMake anteriores), a propriedade isoladamente não as remove. Em uma nova configuração limpa, isso não é um problema.

---

### Etapa H — Avisos que Não Impediram o Build

Os seguintes avisos foram observados durante o processo e **não são erros fatais**:

#### H.1 — CMake CMP0167 / FindBoost

O CMake 4.x gerou um aviso de política relacionado a `FindBoost`. Isso ocorre porque o projeto usa `find_package(Boost REQUIRED)` com a política `CMP0167` que modifica o comportamento de descoberta do Boost em versões mais recentes do CMake.

> **Status:** Aviso, não erro. Não foi necessária nenhuma ação. O build avançou normalmente.

#### H.2 — `-Wodr` / Definições Inconsistentes em Headers FSR-Vulkan

Avisos de ODR (One Definition Rule) podem aparecer quando headers do FSR-Vulkan contêm definições de funções inline ou structs que diferem entre unidades de compilação no contexto do LTO.

> **Status:** Aviso, não erro. Pendente investigação se afeta corretude em runtime.

#### H.3 — `lto-wrapper.exe`: Compilação Serial dos LTRANS

Com `-j 1` (indicado no `build.bat`: `cmake --build ... -- -j 1`) e LTO ativado para `bbgpu`, o `lto-wrapper.exe` processa os objetos LTRANS serialmente. Isso é esperado e apenas significa que a linkagem com LTO é mais lenta.

> **Status:** Informacional. Não é um erro.

---

## 8. Arquivos Modificados e Ajustes Locais

### 8.1 Alterações no Repositório (não commitadas, visíveis em `git diff HEAD`)

| Arquivo                  | Tipo de alteração | Resumo                                                                 | Compartilhável? |
|--------------------------|-------------------|------------------------------------------------------------------------|-----------------|
| `gpu/CMakeLists.txt`     | Modificado        | Substituição do bloco `find_path/find_library` do miniz por `add_subdirectory`; adição de xbyak via `add_subdirectory`; adição de `xbyak::xbyak` em `target_link_libraries`; adição de `set_property(TARGET bbport PROPERTY INTERPROCEDURAL_OPTIMIZATION FALSE)` | **Sim** — deve ser commitado |
| `.gitmodules`            | Modificado        | Adição do submódulo `gpu/third_party/xbyak` → `herumi/xbyak`          | **Sim** — deve ser commitado |

### 8.2 Novos Arquivos/Diretórios no Repositório

| Caminho                          | Origem                                           | Status Git                | Compartilhável? |
|----------------------------------|--------------------------------------------------|---------------------------|-----------------|
| `gpu/third_party/xbyak/`        | Submódulo adicionado (herumi/xbyak v7.43)        | Rastreado (submódulo)      | **Sim** |
| `gpu/third_party/miniz/`        | Clone independente (richgel999/miniz)             | **Não rastreado** (`??`)  | **Pendente** — deveria ser registrado como submódulo |
| `out/libatrac9.a`               | Compilado via `build.bat` / manualmente           | Não rastreado (em `.gitignore` ou ausente do índice) | **Não** — gerado localmente |
| `out/libatrac9_test.a`          | Arquivo intermediário de verificação              | Não rastreado             | **Não** |
| `out/atrac9/`                   | Diretório com `.o` da LibAtrac9                  | Não rastreado             | **Não** |
| `build-log.txt`                 | Log de tentativa de build                        | **Não rastreado** (`??`)  | Opcional |

### 8.3 Ajustes Locais Não Versionados (fora do repositório)

| Item                                                    | Localização                                    | Conteúdo                                  | Reproduzível como? |
|---------------------------------------------------------|------------------------------------------------|-------------------------------------------|--------------------|
| Shim VMA                                                | `C:\msys64\ucrt64\include\vk_mem_alloc.h`      | `#include <vma/vk_mem_alloc.h` (sem `>`) | Criar manualmente com conteúdo correto: `#include <vma/vk_mem_alloc.h>` |
| Headers magic_enum                                      | `C:\msys64\ucrt64\include\magic_enum\*.hpp`    | Headers copiados (não há pacote MSYS2)    | Baixar de https://github.com/Neargye/magic_enum e copiar |
| Variáveis de ambiente no PowerShell                     | Sessão do PowerShell                           | `MINGW_DIR`, `Path`                       | Configurar antes de executar `build.bat` |

---

## 9. Reprodução do Build em Instalação Limpa

> **Aviso:** Esta sequência descreve o processo baseado nos ajustes documentados. Alguns comandos de instalação de pacotes estão marcados como **[pendente de validação]** onde o nome exato do pacote não foi confirmado por `pacman -Si`.

### 9.1 Pré-requisitos

1. Instalar [MSYS2](https://www.msys2.org/) em `C:\msys64`.
2. Abrir o terminal **MSYS2 UCRT64** e instalar os pacotes do sistema:

```bash
pacman -S \
  mingw-w64-ucrt-x86_64-gcc \
  mingw-w64-ucrt-x86_64-cmake \
  mingw-w64-ucrt-x86_64-ninja \
  mingw-w64-ucrt-x86_64-vulkan-headers \
  mingw-w64-ucrt-x86_64-vulkan-loader \
  mingw-w64-ucrt-x86_64-vulkan-memory-allocator \
  mingw-w64-ucrt-x86_64-fmt \
  mingw-w64-ucrt-x86_64-boost \
  mingw-w64-ucrt-x86_64-xxhash \
  mingw-w64-ucrt-x86_64-ffmpeg \
  mingw-w64-ucrt-x86_64-robin-map \
  mingw-w64-ucrt-x86_64-glslang \
  mingw-w64-ucrt-x86_64-zydis \
  mingw-w64-ucrt-x86_64-sdl3 \
  mingw-w64-ucrt-x86_64-pkgconf \
  git
```

> **[pendente de validação]** Os nomes `mingw-w64-ucrt-x86_64-pkgconf` e `mingw-w64-ucrt-x86_64-sdl3` precisam ser verificados com `pacman -Ss <nome>` na versão atual do MSYS2.

3. **Instalar magic_enum manualmente** (sem pacote MSYS2):
```bash
# Dentro do terminal MSYS2 UCRT64
cd /tmp
git clone --depth 1 https://github.com/Neargye/magic_enum.git
cp -r magic_enum/include/magic_enum /ucrt64/include/
```

4. **Criar o shim VMA:**
```bash
# Dentro do terminal MSYS2 UCRT64
echo '#include <vma/vk_mem_alloc.h>' > /ucrt64/include/vk_mem_alloc.h
```

### 9.2 Clonar o Repositório e Inicializar Submódulos

```powershell
git clone https://github.com/Mrsuss60/bloodborne_pc_windows_port.git
cd bloodborne_pc_windows_port
git submodule update --init --recursive
```

> **Atenção:** O diretório `gpu/third_party/miniz/` **não é um submódulo** — precisa ser clonado manualmente:
```powershell
git clone https://github.com/richgel999/miniz.git gpu/third_party/miniz
```
> Ou registrar como submódulo antes do clone:
> ```bash
> git submodule add https://github.com/richgel999/miniz.git gpu/third_party/miniz
> ```

### 9.3 Configurar o Ambiente no PowerShell

```powershell
$env:Path = "C:\msys64\ucrt64\bin;C:\msys64\usr\bin;$env:Path"
$env:MINGW_DIR = "C:\msys64\ucrt64"
```

### 9.4 Executar o Build

```powershell
.\build.bat
```

O script irá:
- Compilar automaticamente `out\libatrac9.a` se não existir.
- Configurar o CMake na primeira execução.
- Compilar `bbgpu`, `bbport` e `bb-gpu-capabilities`.
- Copiar `bbport.exe` para `bb-probe.exe`.
- Executar `scripts/stage_dlls.py`.

### 9.5 Verificação

```powershell
# Confirmar que os binários foram gerados
Test-Path out\bbport.exe
Test-Path out\bb-gpu-capabilities.exe

# Confirmar os símbolos da LibAtrac9
C:\msys64\ucrt64\bin\nm.exe out\libatrac9.a | Select-String "Atrac9"
```

---

## 10. O que foi Comprovado pelo Resultado Final

| Afirmação | Status |
|-----------|--------|
| `out/bbport.exe` foi gerado pelo build | **Confirmado** — arquivo existe |
| `out/bb-gpu-capabilities.exe` foi gerado | **Confirmado** — arquivo existe |
| `out/libatrac9.a` contém todos os 5 símbolos Atrac9 | **Confirmado** — `nm` verificado |
| O jogo abriu no Windows 11 | **Confirmado** pelo usuário — não verificável por auditoria de arquivo |
| VMA_INC resolvido para ucrt64/include | **Confirmado** no CMakeCache |
| MAGIC_ENUM_INC resolvido para ucrt64/include | **Confirmado** no CMakeCache |
| GLSLANG_COMPILER encontrado em ucrt64/bin | **Confirmado** no CMakeCache |
| Todos os compiladores são do UCRT64 | **Confirmado** no CMakeCache |
| `set_property(TARGET bbport PROPERTY INTERPROCEDURAL_OPTIMIZATION FALSE)` presente | **Confirmado** no arquivo atual |

---

## 11. Avisos e Problemas Pendentes

| Item | Categoria | Detalhe |
|------|-----------|---------|
| Shim `vk_mem_alloc.h` sem `>` de fechamento | **Pendente** | O arquivo contém `#include <vma/vk_mem_alloc.h` sem fechar a diretiva. Precisa ser verificado se o compilador aceita ou gera warning. Corrigir para `#include <vma/vk_mem_alloc.h>`. |
| `miniz` não é submódulo | **Pendente** | O diretório `gpu/third_party/miniz/` tem `.git` próprio mas não está em `.gitmodules`. Um clone limpo não o obterá automaticamente. |
| `magic_enum` sem pacote oficial | **Pendente** | Precisa de instalação manual. Documentar a URL e versão exata dos headers utilizados. |
| `C:/msys64/mingw64` hardcoded no `build.bat` | **Pendente** | O script inclui `C:/msys64/mingw64` no `CMAKE_PREFIX_PATH` mesmo quando o ambiente ativo é UCRT64. Em máquinas sem `mingw64` instalado isso é inofensivo; em máquinas onde `mingw64` existe e tem versões incompatíveis das bibliotecas, pode causar conflito de descoberta. |
| `find_program(GLSLANG_COMPILER ... PATHS "C:/msys64/mingw64/bin")` | **Pendente** | O CMakeLists ainda busca o glslang em `C:/msys64/mingw64/bin` antes de usar o PATH. Na máquina atual, o glslang foi encontrado via PATH/CMAKE_PREFIX_PATH antes desse caminho. Em máquinas sem UCRT64 no PATH, o caminho hardcoded para mingw64 seria usado, e poderia falhar. |
| Aviso CMP0167/FindBoost | **Aviso** | Não é erro fatal. Investigar se a política deve ser configurada explicitamente no CMakeLists. |
| Avisos `-Wodr` FSR-Vulkan | **Aviso** | Definições inconsistentes no LTO. Não impediram o build nem a abertura do jogo. Podem indicar UB teórico. |
| Build paralelo limitado a `-j 1` | **Aviso** | O `build.bat` usa `-- -j 1`. O build do `bbgpu` com LTO é serial. Investigar se `-j $(nproc)` com LTO ativado é estável. |
| Log de build não captura erros anteriores | **Informacional** | `build-log.txt` só contém a tentativa com PATH incorreto. Logs de erros de miniz, xbyak e LTO não foram preservados. |

---

## 12. Melhorias Recomendadas para Reprodutibilidade

Em ordem de prioridade:

### 1. Tornar `gpu/third_party/miniz` um Submódulo Oficial

```bash
# No repositório principal:
git submodule add https://github.com/richgel999/miniz.git gpu/third_party/miniz
git commit -m "chore: register miniz as git submodule"
```
Isso garante que `git submodule update --init --recursive` baixe o miniz automaticamente.

### 2. Corrigir o Shim VMA e Integrá-lo ao CMake

Em vez do arquivo manual em `C:\msys64\ucrt64\include\`, modificar o CMake para buscar o header na localização real:
```cmake
find_path(VMA_INC NAMES vma/vk_mem_alloc.h vk_mem_alloc.h
    PATHS "$ENV{MINGW_PREFIX}/include" "C:/msys64/ucrt64/include" "C:/msys64/mingw64/include")
if (VMA_INC)
    target_include_directories(GPUOpen::VulkanMemoryAllocator INTERFACE "${VMA_INC}/vma")
endif()
```
Isso elimina a necessidade do shim e funciona com o pacote MSYS2 como instalado.

### 3. Documentar magic_enum como Dependência Manual

Adicionar no README ou em um script de setup:
- URL: https://github.com/Neargye/magic_enum
- Versão utilizada: (verificar no `magic_enum.hpp`)
- Destino de instalação: `$MSYS2_ROOT/ucrt64/include/magic_enum/`

Alternativa: registrar como submódulo em `gpu/third_party/magic_enum/` e ajustar o CMake para usar `add_subdirectory` ou `find_path` apontando para `third_party/magic_enum/include`.

### 4. Padronizar Caminhos para UCRT64 no CMake e no `build.bat`

- No `build.bat`, adicionar `C:\msys64\ucrt64` **antes** de `C:\msys64\mingw64` no `CMAKE_PREFIX_PATH` (ou remover o fallback para mingw64 quando UCRT64 está em uso).
- No `gpu/CMakeLists.txt`, adicionar `C:/msys64/ucrt64/bin` na lista de PATHS do `find_program(GLSLANG_COMPILER ...)`.

### 5. Validar `libatrac9.a` no `build.bat`

Adicionar verificação após a criação da biblioteca:
```bat
nm out\libatrac9.a | findstr "Atrac9GetHandle" >nul
if errorlevel 1 (
    echo ERRO: out\libatrac9.a nao contem simbolos Atrac9. Recompilando...
    del /f /q out\libatrac9.a
    rem [executar compilacao novamente]
)
```
Também remover `libatrac9.a` se tiver menos de 10.000 bytes, sinalizando arquivo corrompido.

### 6. Investigar Avisos FSR-Vulkan e LTO

Investigar se os avisos `-Wodr` do FSR-Vulkan indicam problemas reais de comportamento indefinido ou são falsos positivos introduzidos pelo LTO.

### 7. Criar Script de Setup de Dependências

Um script PowerShell ou bash que automatize:
1. Instalação dos pacotes MSYS2.
2. Download e cópia dos headers de `magic_enum`.
3. Criação do shim VMA.
4. Clone/inicialização de todos os submódulos (incluindo miniz).

---

## 13. Tabela de Status das Afirmações

| Afirmação | Status | Evidência |
|-----------|--------|-----------|
| O jogo abriu no Windows 11 | **Confirmado** | Relatado pelo usuário |
| `gpu/CMakeLists.txt` foi modificado com 3 adições principais | **Confirmado** | `git diff HEAD` |
| `.gitmodules` foi modificado para adicionar xbyak | **Confirmado** | `git diff HEAD` |
| Xbyak estava ausente do CMakeLists original | **Confirmado** | `git show HEAD:gpu/CMakeLists.txt \| grep xbyak` → vazio |
| miniz não foi encontrado pelo `find_path` original | **Confirmado** | `MINIZ_INC:PATH=MINIZ_INC-NOTFOUND` no CMakeCache |
| VMA está em `vma/vk_mem_alloc.h` no pacote MSYS2 | **Confirmado** | `pacman -Ql mingw-w64-ucrt-x86_64-vulkan-memory-allocator` |
| Shim VMA existe em `C:\msys64\ucrt64\include\vk_mem_alloc.h` | **Confirmado** | Arquivo lido diretamente |
| magic_enum não pertence a nenhum pacote MSYS2 | **Confirmado** | `pacman -Qo` → "Nenhum pacote possui..." |
| `libatrac9.a` contém os 5 símbolos Atrac9 | **Confirmado** | `nm` executado |
| `set_property(TARGET bbport PROPERTY INTERPROCEDURAL_OPTIMIZATION FALSE)` presente | **Confirmado** | Arquivo lido e diff confirmado |
| O compilador usado é do UCRT64 | **Confirmado** | CMakeCache: `CMAKE_CXX_COMPILER: C:/msys64/ucrt64/bin/c++.exe` |
| Versão exata dos headers magic_enum instalados | **Não verificável** | Conteúdo dos arquivos não foi lido |
| Erros de LTO foram causados por assembly inline + `__attribute__((sysv_abi))` | **Inferência** | Padrão de erro consistente; sem log do linker preservado |
| Tentativa anterior de build com PATH incorreto | **Confirmado** | `build-log.txt` |
| miniz não é submódulo git do projeto | **Confirmado** | `git status --short` mostra `?? gpu/third_party/miniz/`; `git ls-files --others` confirma |

---

*Documento gerado em 13 de outubro de 2026 por auditoria do repositório local e do CMakeCache.*
*Baseado exclusivamente em evidências verificáveis: `git diff HEAD`, `git show HEAD:*`, `out/gpu/CMakeCache.txt`, `pacman -Q`/`-Ql`, `nm`, e leitura direta dos arquivos relevantes.*
