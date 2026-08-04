# Tarefas de Triagem de CI

Derivado de `REPORT.md`. Nenhuma tarefa aqui foi executada — este é o plano de ação, nenhum código
foi alterado ainda.

---

### T1 — Instrumentar `Internals.Buffer.Storage.read` para capturar o motivo do `nil`

- **Categoria:** CODE_BUG — investigação (bloqueante para T3/T4)
- **Prioridade:** P0
- **Plataformas afetadas:** iOS, iPadOS, tvOS, watchOS, visionOS, macOS Catalyst
- **Evidência nos logs:** `logs/1_*iPadOS.txt:1802-1804`, `logs/3_*iOS.txt:1824-1827`,
  `logs/5_*visionOS.txt:1807-1809`, `logs/6_*tvOS.txt:1839-1840`, `logs/7_*watchOS.txt:1826-1827` —
  todas com a mensagem `Fatal error: 🐞 RequestDL bug: Buffer reported N readable bytes but
  returned none`.
- **Hipótese de causa:** dentro de `Buffer.Storage.read(at:length:)`
  (`Sources/RequestDL/Internals/Sources/Buffers/Buffer/Internals.Buffer.swift:141-160`), o `guard
  await url.isResourceAvailable() else { return (nil, index) }` (ou a leitura/seek subsequente)
  falha silenciosamente mesmo com um intervalo válido. Não se sabe ainda se é
  `isResourceAvailable()` retornando falso-negativo, um erro engolido pelo `catch { return (nil,
  index) }`, ou esgotamento de recurso (file descriptors) do SO sob alta concorrência.
- **Ação proposta:** adicionar logging temporário (ou um contador `@_spi(Testing)`) em torno do
  `guard`/`catch` de `read(at:length:)` e de `FileStreamBuffer.readData(length:)` para registrar:
  resultado de `isResourceAvailable()`, erro capturado (se houver) e se o descriptor de arquivo
  ainda está aberto no momento da falha. Rodar `fileBuffer_whenRacingImmutable` localmente em loop
  no simulador de iOS até reproduzir e capturar esse log.
- **Validação sugerida:** reproduzir localmente `swift test --filter
  InternalsFileBufferTests/fileBuffer_whenRacingImmutable` num simulador iOS repetidamente (ex.:
  50x) até obter pelo menos uma falha instrumentada; confirmar qual branch do `guard`/`catch` foi
  atingido.

---

### T2 — Testar a hipótese de esgotamento de file descriptors

- **Categoria:** CODE_BUG — investigação
- **Prioridade:** P0
- **Plataformas afetadas:** iOS, iPadOS, tvOS, watchOS, visionOS, macOS Catalyst
- **Evidência nos logs:** mesma de T1; adicionalmente, o fato de `fileBuffer_whenRacingImmutable`
  abrir 1024 tarefas concorrentes contra um único arquivo (`Tests/RequestDLTests/Internals/Sources/
  Buffers/File/InternalsFileBufferTests.swift:788-809`) e de as 6 plataformas afetadas serem todas
  simuladores/sandboxed (tipicamente com `ulimit -n` mais baixo que macOS nativo/Linux CI).
- **Hipótese de causa:** `SystemPackage.FileDescriptor.open` em
  `Internals.FileStreamBuffer.init(readingFrom:)`
  (`Sources/RequestDL/Internals/Sources/Buffers/Buffer/Models/Internals.FileStreamBuffer.swift:60-62`)
  ou chamadas equivalentes via `NIOFileSystem` (`Internals.FileBufferURL`) falham sob pressão de
  descriptors abertos simultaneamente pela suíte inteira rodando em paralelo, e essa falha é
  engolida pelo `catch` em `Buffer.Storage.read`.
- **Ação proposta:** comparar `ulimit -n` do runner macOS nativo vs. o processo de teste no
  simulador iOS (via `Process.self.rlimit` ou `getrlimit` num teste de diagnóstico temporário); se
  for significativamente menor, isso explica por que só simuladores reproduzem.
- **Validação sugerida:** rodar a suíte completa localmente no simulador com `ulimit -n` reduzido
  artificialmente no ambiente macOS nativo e verificar se `fileBuffer_whenRacingImmutable` passa a
  falhar também ali — isso confirmaria ou refutaria a hipótese sem precisar de acesso a um runner
  simulador.

---

### T3 — Corrigir o defeito de leitura concorrente identificado em T1/T2

- **Categoria:** CODE_BUG — correção
- **Prioridade:** P0 (bloqueado por T1)
- **Plataformas afetadas:** iOS, iPadOS, tvOS, watchOS, visionOS, macOS Catalyst
- **Evidência nos logs:** ver T1.
- **Hipótese de causa:** a ser confirmada por T1/T2 antes de qualquer alteração de código.
- **Ação proposta:** implementar o fix mínimo correspondente à causa confirmada (ex.: reabrir/
  retry em `_inputStream` quando o descriptor cacheado estiver inválido; tratar `isResourceAvailable
  () == false` de forma diferente de "arquivo removido"; ou não suprimir o erro de
  `FileDescriptor.open`/`read` dentro do `catch`, propagando-o para diagnóstico em vez de
  silenciá-lo). Não iniciar esta tarefa antes de T1 apontar a causa exata, para evitar corrigir o
  sintoma errado.
- **Validação sugerida:** `fileBuffer_whenRacingImmutable` deve passar de forma consistente
  (ex.: 100 execuções seguidas) num simulador iOS; adicionar um novo teste de regressão que exercite
  especificamente o cenário identificado (ex.: alta contagem de FDs abertos, ou a condição exata de
  `isResourceAvailable()`); depois validar as 6 plataformas via CI real.

---

### T4 — Adicionar teste de regressão para o cenário fatal de `BodySequence`

- **Categoria:** test-coverage
- **Prioridade:** P1 (após T3)
- **Plataformas afetadas:** iOS, iPadOS, tvOS, watchOS, visionOS
- **Evidência nos logs:** o crash ocorre com `InternalsBodySequenceTests` já `passed` e
  `InternalsFileBufferTests` ainda em execução (`logs/1_*iPadOS.txt:1724,1802`; mesmo padrão em
  `3_*iOS.txt:1773,1824`, `5_*visionOS.txt:1731,1807`, `6_*tvOS.txt:1763,1839`,
  `7_*watchOS.txt:1770,1826`), indicando que o gatilho fatal vem de execução concorrente entre
  suítes, não de um teste isolado de `BodySequence`.
- **Hipótese de causa:** não há hoje um teste que force `Internals.BodySequence` a ler de um
  `Internals.Buffer` compartilhado sob alta concorrência de outras operações de I/O simultâneas.
- **Ação proposta:** escrever um teste que combine leitura de `BodySequence` com pressão
  concorrente equivalente à de `fileBuffer_whenRacingImmutable`, para que o cenário fatal tenha uma
  cobertura direta (hoje só é coberto indiretamente pela execução paralela da suíte inteira).
- **Validação sugerida:** o novo teste deve falhar de forma determinística antes do fix de T3 e
  passar de forma consistente depois.

---

### T5 — Corrigir extração de `.profdata` no job 🍎 macOS para o layout do Xcode 26

- **Status:** ✅ Corrigido em `.github/workflows/swift-ci.yaml` (jobs `apple-tests` e
  `coverage-upload`). Confirmado empiricamente numa máquina local com Xcode 26.6: o job
  `apple-tests` agora extrai a cobertura por linha diretamente do `.xcresult` via
  `xcrun xccov view --archive` (evita depender do layout interno da derived data), e o job
  `coverage-upload` converte esse output para `.lcov` com um script Python embutido, mantendo o
  caminho antigo (`.profdata` + `.xctest` + `llvm-cov export`) como fallback para os artefatos de
  `third-party-tests` (Linux via SwiftPM), que não são afetados por essa mudança do Xcode.
- **Categoria:** CODE_BUG — script de CI
- **Prioridade:** P1
- **Plataformas afetadas:** Coverage Upload (consumidor), job 🍎 macOS (produtor do artefato)
- **Evidência nos logs:** `logs/0_*Coverage Upload.txt:207-213` (`Nenhum .profdata em
  artifacts/apple-macOS/` → `exit code 1`); `logs/8_*macOS.txt:1950-1953` (`find .xcbuild -name
  "*.profdata"` não encontra nada apesar de `-enableCodeCoverage YES` em `8_*macOS.txt:539` e um
  `.xcresult` válido gerado em `8_*macOS.txt:1948`).
- **Hipótese de causa:** na imagem `macos-26-arm64` / Xcode 26.6 usada neste run
  (`logs/0_*.txt:11-17`), os dados de cobertura passaram a residir dentro do pacote `.xcresult` em
  vez de como `.profdata` solto sob `.xcbuild`, quebrando a extração baseada em `find`.
- **Ação proposta:** trocar a extração por `xcrun xcresulttool export` (ou `xccov`) apontando para
  o `.xcresult` gerado, ou localizar o `.profdata` dentro do próprio pacote `.xcresult`
  (`*.xcresult/**/*.profdata`) em vez de `.xcbuild` diretamente. Esta mudança vive no workflow
  reutilizável `request-dl/.github` (`Uses: request-dl/.github/.github/workflows/swift-ci.yaml`,
  `logs/0_*.txt:31`), não neste repositório — abrir a alteração lá.
- **Validação sugerida:** rodar o job 🍎 macOS e confirmar que `coverage-artifacts/` contém um
  `.profdata` não vazio antes do upload; depois confirmar que o job Coverage Upload processa
  `apple-macOS` sem o erro `Nenhum .profdata`.

---

### T6 — Adicionar smoke check ao workflow para artefato de cobertura vazio

- **Status:** ✅ Corrigido em `.github/workflows/swift-ci.yaml` (job `apple-tests`, passo
  `🔀 Extract Coverage Report`). Após gerar `coverage-artifacts/coverage.xccov`, o passo agora
  falha explicitamente (`exit 1`, sem `|| true`) se o arquivo estiver ausente ou vazio, em vez de
  deixar a falha só aparecer depois no job Coverage Upload. Validado localmente simulando (1) um
  `.xcresult` corrompido/inexistente — já falha antes por causa do `xccov` sob `-e`/`pipefail` — e
  (2) um `coverage.xccov` vazio gerado com sucesso — capturado pelo novo `[ ! -s ... ]`. Escopo
  mantido no job 🍎 macOS, conforme a ação proposta original; o job `third-party-tests` (Linux) não
  foi alterado por não fazer parte da evidência do incidente.
- **Categoria:** test-coverage — CI
- **Prioridade:** P2 (após T5)
- **Plataformas afetadas:** Coverage Upload
- **Evidência nos logs:** o `|| true` em `logs/8_*macOS.txt:1952` engoliu silenciosamente a falha
  de extração; o problema só apareceu 7 jobs depois, no Coverage Upload, tornando o diagnóstico
  mais lento do que precisaria ser.
- **Hipótese de causa:** falha de descoberta de artefato (ex.: mudança futura de layout do Xcode)
  volta a passar despercebida no job produtor porque o script usa `|| true`.
- **Ação proposta:** no job 🍎 macOS, falhar explicitamente (sem `|| true`) se
  `coverage-artifacts/` não contiver nenhum `.profdata` após a extração, em vez de deixar o erro
  surgir só no Coverage Upload.
- **Validação sugerida:** simular localmente um `.xcresult` sem `.profdata` acessível e confirmar
  que o job 🍎 macOS falha imediatamente com uma mensagem clara, em vez de subir um artefato
  incompleto.

---

### T7 — Anexar nome do teste em execução ao crash de `Fatal error` nos runners Apple

- **Status:** ✅ Corrigido em `.github/workflows/swift-ci.yaml` (job `apple-tests`). O passo
  `🧪 Run Tests with Coverage` agora grava a saída bruta do `xcodebuild` em `raw-test-output.log`
  via `tee` (antes ela só passava pelo `xcbeautify`, cujo output "bonito" não sobrevive
  de forma confiável a um crash abrupto). Um novo passo `🚨 Identify Test Running at Crash Time`
  (`if: failure()`) faz o parsing desse log comparando linhas de `started` com linhas de
  `passed`/`failed`/`skipped` (cobrindo tanto o formato clássico do XCTest quanto o do
  swift-testing) e emite uma anotação `::error::` listando exatamente qual(is) teste(s)/suíte(s)
  ainda estavam em execução no momento da falha — eliminando a necessidade de inferir isso
  manualmente comparando `Suite ... started` com a ausência de `passed/failed`. O log bruto
  também é sempre enviado como artefato (`raw-test-log-<platform>`) para inspeção posterior.
  Validado localmente simulando o parsing em um log sintético com um `Fatal error` no meio de um
  teste em andamento: o script aponta corretamente a suíte e o teste ainda abertos.
- **Categoria:** efficiency — observabilidade de CI
- **Prioridade:** P3
- **Plataformas afetadas:** iOS, iPadOS, tvOS, watchOS, visionOS
- **Evidência nos logs:** em nenhum dos 5 logs de crash a linha do `Fatal error` é precedida por
  informação de qual teste/suíte estava ativo naquele exato momento (só foi possível inferir
  indiretamente comparando `Suite ... started` vs. ausência de `Suite ... passed/failed`).
- **Hipótese de causa:** não é um bug, é uma lacuna de diagnóstico — `xcodebuild`/swift-testing não
  está configurado para emitir um resumo de "testes em execução no momento do crash" nestes
  runners.
- **Ação proposta:** avaliar habilitar `-resultBundlePath` mais granular ou o modo de teste
  paralelo com relatório por-teste, para que uma futura triagem não precise inferir qual teste
  disparou o crash a partir de ausência de linhas de log.
- **Validação sugerida:** provocar um crash controlado localmente e confirmar que o novo log
  identifica o teste responsável diretamente.

---

## Resumo de prioridades

| ID | Prioridade | Bloqueado por |
|---|---|---|
| T1 | P0 | — |
| T2 | P0 | — (pode rodar em paralelo a T1) |
| T3 | P0 | T1, T2 |
| T4 | P1 | T3 |
| T5 | P1 | — |
| T6 | P2 | T5 |
| T7 | P3 | — |
