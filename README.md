# Leitor de áudio

Transforma documentos (PDF, DOCX, TXT, MD) em narrações em áudio, com escolha de **voz**, **estilo** (História, Artigo, HQ, Mangá, Documentário) e **velocidade**.

- **Backend:** Spring Boot 3.4, Java 17, JPA/Hibernate, PostgreSQL, Spring Security + JWT
- **Frontend:** Next.js 15 (App Router), React 19, TypeScript, Tailwind CSS 3
- **Voz:** API de áudio da OpenAI (atrás de uma interface `TtsClient`, fácil de trocar)

## Como rodar

### 1. Banco de dados
```bash
docker compose up -d        # PostgreSQL 16 em localhost:5432 (db/usuário/senha: leitor_audio / leitor / leitor)
```

### 2. Backend
```bash
cd backend
export OPENAI_API_KEY=sk-...        # sem a chave tudo funciona, mas as narrações terminam em ERRO com uma mensagem clara
export JWT_SECRET="uma-string-aleatoria-com-mais-de-32-caracteres"
mvn spring-boot:run                  # http://localhost:8080
```

Variáveis de ambiente aceitas (todas têm padrão para desenvolvimento):

| Variável | Padrão | Para quê |
|---|---|---|
| `DB_URL`, `DB_USER`, `DB_PASSWORD` | localhost / leitor / leitor | Conexão com o PostgreSQL |
| `DDL_AUTO` | `update` | Em produção use `validate` e um migrador (Flyway/Liquibase) |
| `JWT_SECRET` | valor de exemplo | **Troque.** Mínimo 32 caracteres |
| `JWT_EXPIRATION_MINUTES` | `1440` | Validade do token (24 h) |
| `AUDIO_DIR` | `./storage/audios` | Onde os `.mp3` são gravados |
| `CORS_ORIGINS` | `http://localhost:3000` | Origens do frontend (separe por vírgula) |
| `OPENAI_API_KEY` | vazio | Chave da OpenAI |
| `OPENAI_TTS_MODEL` | `gpt-4o-mini-tts` | Ou `tts-1` / `tts-1-hd` (ver "Velocidade") |
| `TTS_MAX_CHARS` | `60000` | Limite de caracteres por narração (controla custo) |

### 3. Frontend
```bash
cd frontend
npm install
cp .env.example .env.local           # NEXT_PUBLIC_API_URL=http://localhost:8080
npm run dev                          # http://localhost:3000
```

## Estrutura do backend

```
dan.leitor_audio
├── config/       AppProperties (app.*), SecurityConfig (JWT, CORS, BCrypt), AsyncConfig (pool das narrações)
├── security/     JwtService, JwtAuthenticationFilter, UsuarioPrincipal
├── model/        Usuario, Documento, Narracao + enums Voz, EstiloNarracao, StatusNarracao
├── repository/   Spring Data JPA
├── dto/          records de entrada e saída (a entidade nunca vai direto para o JSON)
├── service/      AuthService, DocumentoService, NarracaoService,
│                 TextExtractorService (PDFBox/POI), TextoChunker, AudioStorageService,
│                 NarracaoProcessor (geração assíncrona)
├── tts/          TtsClient (interface), OpenAiTtsClient
├── controller/   AuthController, DocumentoController, NarracaoController, OpcoesController
└── exception/    ApiException, ErrorResponse, GlobalExceptionHandler
```

## API

Todas as rotas, exceto `/api/auth/registrar` e `/api/auth/login`, exigem `Authorization: Bearer <token>`. Cada usuário só enxerga os próprios documentos e narrações (acessar o de outra pessoa devolve 404).

| Método | Rota | Descrição |
|---|---|---|
| POST | `/api/auth/registrar` | `{nome, email, senha}` → `{token, usuario, ...}` (201) |
| POST | `/api/auth/login` | `{email, senha}` → `{token, usuario, ...}` |
| GET | `/api/auth/me` | Usuário logado |
| GET | `/api/opcoes` | Vozes, estilos e limites de velocidade (valor + rótulo em português) |
| POST | `/api/documentos` | `multipart/form-data`, campo `arquivo` (PDF/DOCX/TXT/MD, até 20 MB) → 201 |
| GET | `/api/documentos` | Lista (sem o texto, para ser leve) |
| GET | `/api/documentos/{id}` | Detalhe com `textoExtraido` |
| DELETE | `/api/documentos/{id}` | Apaga o documento, as narrações e os arquivos de áudio |
| POST | `/api/narracoes` | `{documentoId, voz, estiloNarracao, velocidade?}` → **202**, status `PENDENTE` |
| GET | `/api/narracoes?documentoId=` | Lista (filtro opcional) |
| GET | `/api/narracoes/{id}` | Detalhe/status |
| GET | `/api/narracoes/{id}/audio?download=` | Stream do MP3 (aceita `Range`) |
| POST | `/api/narracoes/{id}/reprocessar` | Tenta de novo uma narração com `ERRO` |
| DELETE | `/api/narracoes/{id}` | Apaga a narração e o arquivo |

Erros seguem o formato `{status, mensagem, erros?: {campo: mensagem}, timestamp}`.

## Como a narração é gerada

1. `POST /api/narracoes` grava a narração como `PENDENTE` e publica um evento.
2. Depois do **commit**, o `NarracaoProcessor` (em thread separada) muda para `EM_ANDAMENTO`.
3. O texto é dividido em trechos de ~3.900 caracteres (em fim de parágrafo/frase), pois a API aceita até 4.096 por chamada.
4. Cada trecho vira MP3 e é gravado em sequência em `{AUDIO_DIR}/{usuarioId}/{narracaoId}.mp3`.
5. Termina em `CONCLUIDA` (com o caminho do arquivo) ou `ERRO` (com `mensagemErro`). O frontend consulta o status a cada 3 s enquanto houver narração em andamento.
6. Se o servidor reiniciar no meio, as narrações presas viram `ERRO` e podem ser reprocessadas pelo botão "Tentar de novo".

**Estilo e voz:** com `gpt-4o-mini-tts`, o estilo (História, Mangá…) e o timbre (Grave, Heroica…) são enviados como `instructions` em português, e a voz base é escolhida entre as vozes da OpenAI (`Voz` → `echo`, `nova`, `onyx`, `shimmer`, `fable`, `sage`). A tabela de mapeamento fica em `OpenAiTtsClient`.

**Velocidade:** `tts-1`/`tts-1-hd` aplicam a velocidade no próprio áudio. `gpt-4o-mini-tts` não aceita esse parâmetro, então a API devolve `velocidadeAplicadaNoAudio: false` e o **player do frontend** aplica a velocidade escolhida (e permite mudar depois). Se você precisa da velocidade gravada no arquivo, use `OPENAI_TTS_MODEL=tts-1`; nesse caso o estilo não é aplicado, pois esse modelo não aceita `instructions`.

## Mudanças que fiz nas suas entidades

| Onde | Problema | Correção |
|---|---|---|
| `Usuario.narracoes` | Era `List<StatusNarracao>` (um enum) com `@OneToMany` | `List<Narracao>` |
| `Documento.estilosNarracao` | Era `List<EstiloNarracao>` (enum) com `mappedBy = "documento"` | `List<Narracao> narracoes` |
| `Usuario.senha` | `@Size(max = 20)` na entidade rejeitaria o hash BCrypt (60 caracteres) ao salvar | Removido da entidade; o limite 7–20 é validado no DTO `RegistroRequest` |
| `Narracao` | Sem onde guardar o motivo de falha | Novo campo `mensagemErro` |

Os enums `Voz`, `EstiloNarracao`, `StatusNarracao` e a classe `LeitorAudioApplication` ficaram **idênticos** aos seus. O `application.properties` manteve suas configurações de upload e ganhou as demais.

## Decisões e limites conhecidos

- **Token no `localStorage`:** simples e funciona bem em desenvolvimento, mas fica exposto a XSS. Para produção, considere cookie `HttpOnly` + `SameSite`.
- **Sem OCR:** PDFs escaneados (só imagem) são recusados com uma mensagem explicando o motivo.
- **Custo:** cada narração chama a API paga da OpenAI. `TTS_MAX_CHARS` limita o tamanho; não há cota por usuário ainda.
- **Fila em memória:** o pool de 2–4 threads é suficiente para começar; para escala, troque por uma fila externa (RabbitMQ/SQS) e armazenamento em S3.
- **Áudio no disco local:** em mais de uma instância, use storage compartilhado.
- **MP3 concatenado:** os trechos são unidos byte a byte; a maioria dos players lida bem, e a duração exibida pode ter pequena imprecisão.
- **`ddl-auto=update`:** prático para desenvolver, mas não substitui migrações versionadas em produção.

## Testes

`TextoChunkerTest` e `TextExtractorServiceTest` (JUnit, sem Spring): `mvn test`.
