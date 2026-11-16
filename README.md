# Bold Pear

Bem-estar mental para astronautas — NASA Space Apps Challenge 2026.
A Bold Pear, uma pera 3D, conduz em voz alta um check-in com PHQ-9 e GAD-7 nos idiomas
português, inglês, alemão, francês e japonês, e mostra a evolução num dashboard.

## Rodar

```bash
flutter pub get
flutter run -d chrome        # web
flutter run                  # Android / iOS
```

Build web: `flutter build web --release --no-web-resources-cdn`

O app já aponta para o projeto Supabase **Bold Pear** (`omhfegwzorvrpscdftvg`, região sa-east-1).
Para usar outro projeto:
`--dart-define=SUPABASE_URL=https://xxx.supabase.co --dart-define=SUPABASE_KEY=sb_publishable_...`

## Supabase

- Tabelas `profiles` e `checkins`, com RLS: cada pessoa só vê e grava os próprios dados.
- Um gatilho cria o perfil no cadastro (nome vem de `display_name`).
- SQL em `supabase/migrations/`.
- **Confirmação de e-mail:** projetos novos exigem confirmar o e-mail antes do primeiro login, e o
  SMTP padrão do Supabase envia poucos e-mails por hora. Para a demo, dá para desligar em
  Authentication → Sign In / Providers → Email → *Confirm email*, ou configurar um SMTP próprio.

## Painel NASA (admin)

### Acesso de demonstração

| E-mail | Senha |
|---|---|
| `admin.nasa@nasa.com` | `Nasa2026` |

Esta conta abre o **painel de administração com todos os astronautas**: ao entrar com ela, o
app vai direto para `/admin`, com os indicadores de toda a tripulação e a ficha completa de cada
astronauta (perfil, check-ins, pontuações de PHQ-9/GAD-7, alertas e histórico). O acesso é só de
leitura. Para ver o app como astronauta, crie uma conta comum pelo cadastro.

### Como funciona

Contas listadas em `public.admins` entram direto no painel `/admin` (sem pera, onboarding ou
check-in). O painel mostra os indicadores da tripulação (ativos, atrasados, em atenção, item 9,
médias de PHQ-9/GAD-7, adesão, faixas, check-ins por semana, sintomas mais marcados) e a ficha
de cada astronauta (perfil, estatísticas, alertas, evolução, itens e histórico completo).

1. Rode `supabase/migrations/20261115230000_admin_dashboard.sql` no SQL Editor.
2. Crie a conta (cadastro no app ou Authentication → Users → Add user).
3. Troque o e-mail em `supabase/grant_admin.sql` e rode no SQL Editor — ou, sem abrir o painel
   do Supabase, com um [token pessoal](https://supabase.com/dashboard/account/tokens):
   ```bash
   SUPABASE_ACCESS_TOKEN=sbp_... python scripts/grant_admin.py admin@exemplo.com   # concede
   python scripts/grant_admin.py --revoke admin@exemplo.com                        # remove
   python scripts/grant_admin.py --list                                            # lista
   ```

Testes do painel: `flutter test test/admin_load_test.dart` (1.000 astronautas / ~15 mil
check-ins, paginação da API, casos graves e telas em celular e desktop).

Só o banco decide quem é admin (`public.is_admin()` + RLS, somente leitura); o app não consegue
se promover.

## Voz neural

A voz da pera (Supertonic 3 int8 via sherpa-onnx, OpenRAIL-M, ~138 MB) **não vai no app nem
no repositório**: é baixada na primeira abertura, numa tela de carregamento antes da escolha
de idioma. Os arquivos vêm do espelho do sherpa-onnx no Hugging Face, numa revisão fixada
(`Env.voiceModelUrl`); cada um é conferido por tamanho e SHA-256, e o download retoma de onde
parou se a conexão cair. Se falhar, dá para tentar de novo ou seguir com a voz do sistema
(e baixar depois em Ajustes). Na web, usa a voz do navegador. Outra origem:
`--dart-define=VOICE_MODEL_URL=https://.../pasta`.

## Fluxo

0. Primeira abertura (celular): download da voz da pera, com barra de progresso.
1. Boas-vindas: escolha do idioma (a pera cumprimenta), criar conta ou entrar.
2. Cadastro (nome, e-mail, senha) ou login; recuperação de senha por e-mail.
3. Primeiro acesso: a pera avisa que é só um programa e que ninguém ouve ao vivo; escolha da voz
   (feminina/masculina) e do nome; forma de tratamento em português e francês.
4. Dashboard: próximo check-in (a cada 14 dias), gráficos de PHQ-9 e GAD-7 sobre as faixas de
   gravidade, histórico e respiração guiada.
5. Check-in falado: PHQ-9 → impacto no dia a dia (se houver sintoma) → GAD-7. As perguntas são
   faladas; a tela mostra só as opções. Legendas opcionais (botão CC).
6. Resultado: faixas, aviso de segurança se o item 9 for positivo, práticas sugeridas.

A pera fica viva em todas as telas (`lib/app/shell.dart`): o mesmo HTML com three.js
(`assets/avatar/index.html`, offline) roda numa WebView no celular e num iframe na web.

## Estrutura

```
lib/
  app/            estado da sessão, rotas (go_router) e a moldura com a pera
  core/           cores (pi_cores.dart), tema, textos nos 5 idiomas, widgets
  features/
    auth/         boas-vindas, login, cadastro
    profile/      onboarding, ajustes, repositório do perfil
    checkin/      escalas, pontuação, fluxo falado, resultado, respiração
    dashboard/    evolução e histórico
    avatar/       pera 3D (WebView / iframe) e seu controlador
    voice/        voz (TTS) que move a boca da pera e as legendas
```

## Pendências antes de uso real

- Trocar os textos de GAD-7 (todos os idiomas) e PHQ-9 (DE, FR, JA) pelas traduções oficiais validadas.
- Escolher as vozes neurais por idioma e gênero e pré-gerar os áudios (hoje usa a voz do sistema).
- Ferramenta de bem-estar; não faz diagnóstico.
