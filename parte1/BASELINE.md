# Baseline - Parte 1

## Repositórios

- Meu fork: https://github.com/rafaelmax57/flaskbb
- Fork do professor (adicionei como remoto `upstream-prof`): https://github.com/jeffsantos/flaskbb
- Commit base: `22989b72419d4083989cc6123be69228607ac7b1` ("a couple of fixes around redis cacheing"), que é o mesmo que está no guia de setup.

## Ambiente

- Windows 11 Pro
- Python 3.13.15
- uv 0.12.19
- pytest 9.0.3 (com pytest-cov 7.1.0, pytest-mock 3.15.1 e pytest-xdist 3.8.0)

Instalei as dependências com `uv sync`.

## Problemas que tive

Na primeira vez que rodei `uv run pytest` deu 74 passed, 158 errors e 1 failed. Não mexi em nenhum código do flaskbb, os dois problemas eram de ambiente.

**1) FileExistsError na pasta instance**

Os 158 erros eram todos `FileExistsError: [WinError 183]` na hora de criar a pasta `flaskbb/instance`. Os testes rodam em paralelo (o pyproject tem `--numprocesses auto`, do xdist) e o `app.py` (linhas 129-130) primeiro verifica se a pasta existe e depois cria. Como vários processos começam juntos, mais de um tenta criar a pasta e os outros quebram.

Resolvi criando a pasta antes de rodar:

```powershell
mkdir instance
```

Depois disso ficou 231 passed, 1 skipped e 1 failed.

**2) Teste de tradução falhando**

O `tests/unit/utils/test_translations.py::test_flaskbbdomain_translations` falhava porque `domain.get_translations()` voltava um `NullTranslations`. O repositório só tem os arquivos `.po`, os `.mo` compilados não vêm junto. Resolvi compilando:

```powershell
uv run flaskbb translations compile
```

## Resultado dos testes

```powershell
uv run pytest
```

Final da saída:

```text
[gw2] [100%] PASSED tests/unit/utils/test_translations.py::test_flaskbbdomain_translations

============================= 233 passed in 13.28s ==============================
```

233 testes passando, nenhuma falha.

## Cobertura inicial

Rodei a cobertura dos três módulos juntos (com branch):

```powershell
uv run pytest --cov=flaskbb.forum --cov=flaskbb.management --cov=flaskbb.user --cov-branch --cov-report=term-missing
```

Deu 233 passed e 25% no total. Depois tirei o relatório de cada módulo separado usando o mesmo `.coverage`:

```powershell
uv run coverage report --include="flaskbb/forum/*" --show-missing
uv run coverage report --include="flaskbb/management/*" --show-missing
uv run coverage report --include="flaskbb/user/*" --show-missing
```

Resumo:

| Módulo | Cobertura |
|---|---|
| forum | 34% |
| management | 7% |
| user | 34% |

### forum

```text
Name                        Stmts   Miss Branch BrPart  Cover   Missing
-----------------------------------------------------------------------
flaskbb\forum\__init__.py       2      2      0      0     0%   1-3
flaskbb\forum\forms.py         91     91     16      0     0%   12-212
flaskbb\forum\locals.py        29     18      8      2    46%   11-23, 27-28, 34-35, 41-42, 44, 48, 55-58
flaskbb\forum\models.py       641    271    150     33    60%   12-201, 219-256, 268->271, 282-294, 312->exit, 320->337, 342-343, 347-348, 358-359, 361, 373-374, 376, 388, 391->421, 414-418, 423, 432, 454, 479, 491->exit, 497->exit, 508-575, 582-583, 587-588, 591, 594, 605->608, 612-613, 616, 620-688, 714-715, 734, 784, 791-800, 828-829, 852-853, 868->872, 884-885, 898->901, 904-905, 911, 922, 925, 936, 951-957, 965, 969, 991, 1016, 1017->exit, 1028, 1035-1037, 1053-1123, 1127-1128, 1131, 1134-1146, 1177, 1202->1208, 1275, 1305-1306, 1317->1326, 1331-1332, 1354-1367, 1388, 1397, 1402-1403, 1449-1488, 1525-1526, 1594-1595, 1663
flaskbb\forum\utils.py         10      7      2      1    33%   12-25, 30->exit, 34
flaskbb\forum\views.py        499    465    108      0     6%   13-1165
-----------------------------------------------------------------------
TOTAL                        1272    854    284     36    34%
```

### management

```text
Name                             Stmts   Miss Branch BrPart  Cover   Missing
----------------------------------------------------------------------------
flaskbb\management\__init__.py       4      4      0      0     0%   13-20
flaskbb\management\forms.py        208    208     50      0     0%   12-516
flaskbb\management\models.py        66     50     12      2    28%   12-77, 96-118, 130-133, 141, 147-148
flaskbb\management\plugins.py       16     10      2      0    44%   1-10, 38-46
flaskbb\management\views.py        543    502    134      1     6%   12-1171, 1179, 1210-1349, 1356-1357
----------------------------------------------------------------------------
TOTAL                              837    774    198      3     7%
```

### user

```text
Name                                  Stmts   Miss Branch BrPart  Cover   Missing
---------------------------------------------------------------------------------
flaskbb\user\__init__.py                  4      4      0      0     0%   12-19
flaskbb\user\forms.py                    45     36      2      0    23%   12-52, 56-78, 83, 87-102, 108-118, 122
flaskbb\user\models.py                  246    193     46      8    24%   12-98, 103-191, 200, 204-214, 218-219, 223-224, 228-257, 263, 267, 270, 274-276, 280, 283, 291-333, 338->exit, 342, 348->exit, 352, 360-389, 393-394, 397, 410-454, 463-477, 483-494, 497-498, 501-502, 507-508, 511, 524-527
flaskbb\user\plugins.py                  26     23      0      0    12%   12-32, 49-87
flaskbb\user\services\__init__.py         0      0      0      0   100%
flaskbb\user\services\factories.py       33     33      2      0     0%   13-89
flaskbb\user\services\update.py          42     27      0      0    36%   12-29, 38-48, 55-65, 74-83
flaskbb\user\services\validators.py      40     21     14      0    61%   11-30, 43-48, 53-59, 64-70, 75-81, 86-98
flaskbb\user\views.py                   124     51      8      0    61%   13-49, 52, 70, 73, 77-83, 86, 104, 107, 111-117, 120, 138, 141, 145-151, 154, 172, 175, 201-202
---------------------------------------------------------------------------------
TOTAL                                   560    388     72      8    34%
```

## Observações

- Algumas linhas do começo dos arquivos aparecem como não cobertas (tipo `forum/__init__.py` 1-3). Pelo que entendi é código que roda no import, antes do coverage começar a medir. Por isso vou usar o mesmo comando daqui na comparação final.
- Os arquivos com a saída completa (`baseline_pytest.txt` e `baseline_cov.txt`) ficaram só no meu computador, não subi pro fork.
