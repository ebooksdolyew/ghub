# ✅ VALIDAÇÃO JORNADA PARA 12×36

## O que foi implementado

**Nova regra de validação EXCLUSIVA para escala 12×36:**

Para funcionários em escala 12×36, um ATM só é contabilizado se a coluna **JORNADA** estiver preenchida na mesma linha.

---

## Regras

### 1. Extração (readAtmsFromTable)
- Extrai **TODOS** os ATMs encontrados em ENT1, **todos os dias da semana** (Seg-Dom)
- **NÃO rejeita** fim de semana para 12×36
- Rastreia a linha (rowY) de cada ATM para validação posterior

### 2. Validação (validateAtm12x36ByJornada)
- Para cada ATM extraído, verifica célula **JORNADA** na mesma linha
- **Se JORNADA vazia:** ❌ Rejeita (não contabiliza)
- **Se JORNADA preenchida:** ✅ Mantém (contabiliza)

### 3. Processamento (processPage)
- Para 12×36: aplica validação JORNADA **DEPOIS** da extração
- Para Convencional: sem mudança (continua rejeitando Sáb/Dom)

---

## Fluxo de Processamento

### Funcionário Convencional
```
readAtmsFromTable()
├─ Para cada dia:
│  ├─ Seg-Sex → MANTÉM
│  └─ Sáb-Dom → PULA ❌
└─ Retorna apenas Seg-Sex

processPage()
└─ Sem validação adicional (já validado)

Resultado: apenas ATMs de dias úteis
```

### Funcionário 12×36 (NOVO)
```
readAtmsFromTable()
├─ Para cada dia (Seg-Dom):
│  └─ MANTÉM TODOS (sem rejeição por dia)
└─ Retorna todos + rastreia rowY

processPage()
└─ Chama validateAtm12x36ByJornada()
   ├─ Para cada ATM:
   │  ├─ Verifica JORNADA na mesma linha
   │  ├─ JORNADA vazia → REMOVE ❌
   │  └─ JORNADA preenchida → MANTÉM ✅
   └─ Retorna apenas ATMs com JORNADA válida

Resultado: ATMs com vínculo válido JORNADA
```

---

## Exemplo Prático

### Dados no PDF (12×36)
```
| Data  | ENT1 | JORNADA     | Situação                     |
|-------|------|-------------|------------------------------|
| 20/05 | ATM  | 12x36       | ✅ CONTABILIZAR              |
| 21/05 | ATM  | (vazio)     | ❌ DESCONSIDERAR (sem JORNADA)|
| 22/05 | ATM  | Folga       | ✅ CONTABILIZAR              |
| 23/05 | ATM  | 12x36       | ✅ CONTABILIZAR (sábado OK)  |
| 24/05 | ATM  | (vazio)     | ❌ DESCONSIDERAR (sem JORNADA)|
```

### Processamento
```
Etapa 1 - Extração (readAtmsFromTable):
  Encontrados: 5 ATMs (20, 21, 22, 23, 24)
  rowYByKey = {
    "20/05/2026 (Sex)": Y1,
    "21/05/2026 (Sab)": Y2,
    "22/05/2026 (Dom)": Y3,
    "23/05/2026 (Seg)": Y4,
    "24/05/2026 (Ter)": Y5
  }

Etapa 2 - Validação JORNADA (validateAtm12x36ByJornada):
  20/05: JORNADA="12x36" → ✅ MANTÉM
  21/05: JORNADA="" → ❌ REMOVE
  22/05: JORNADA="Folga" → ✅ MANTÉM
  23/05: JORNADA="12x36" → ✅ MANTÉM
  24/05: JORNADA="" → ❌ REMOVE

Resultado final: 3 ATMs válidos (20, 22, 23)
```

---

## Isolamento das Regras

### Convencional (Seg-Sex)
```javascript
// Em readAtmsFromTable():
if (!is12) { // Convencional
  if (dow === 'SAB' || dow === 'DOM') {
    continue; // REJEITA SÁB/DOM AQUI
  }
}
```

**Não afeta:** 12×36, ENT2, SAI1, SAI2, ou qualquer outra coluna

### 12×36 (JORNADA)
```javascript
// Em processPage():
if (is12 && tableStruct) {
  atmDays = validateAtm12x36ByJornada(atmDays, tableStruct, rowYByKey);
  // VALIDA JORNADA APENAS AQUI
}
```

**Não afeta:** Convencional, dia da semana, ou qualquer outra validação

---

## Dados Técnicos

### Funções Envolvidas

| Função | Responsabilidade |
|--------|------------------|
| `readAtmsFromTable()` | Extrai TODOS os ATMs (Seg-Dom para 12×36), rastreia rowY |
| `validateAtm12x36ByJornada()` | Filtra ATMs 12×36 por JORNADA preenchida |
| `detectAtmByRow()` | Retorna {atmDays, rowYByKey, tableStruct} |
| `processPage()` | Aplica validação JORNADA para 12×36 apenas |

### Mudanças de Interface

```javascript
// readAtmsFromTable ANTES:
return atmDays; // Array

// readAtmsFromTable DEPOIS:
return { atmDays, rowYByKey }; // Object com metadados

// detectAtmByRow ANTES:
return atmDays; // Array

// detectAtmByRow DEPOIS:
return { atmDays, rowYByKey, tableStruct }; // Object com tudo

// Compatibilidade:
const atmDays = detectionResult.atmDays || detectionResult;
// Lida com ambos os formatos
```

---

## Teste Prático

### Cenário 1: Convencional (sem mudança)
```
Input: PDF com Convencional
Atestados: Seg, Ter, Qua, Sab, Dom
Processamento: Rejeita Sab/Dom
Saída: 3 ATMs (Seg, Ter, Qua)
```

### Cenário 2: 12×36 (nova regra)
```
Input: PDF com 12×36
Atestados: Seg(JORNADA="12×36"), Ter(vazio), Qua(JORNADA="Folga")
Processamento: Extrai 3, valida 2 (Ter=vazio → rejeita)
Saída: 2 ATMs (Seg, Qua)
```

---

## Garantias

✅ **Sem interferência entre as regras**
- Convencional continua rejeitando Sáb/Dom como antes
- 12×36 valida apenas JORNADA (novidade)

✅ **Sem quebra de compatibilidade**
- CSV exporta dados corretos automaticamente
- Relatório atualizado com contexto

✅ **Totalmente isolado**
- Validação JORNADA ocorre **DEPOIS** da extração
- Não afeta nenhum outro cálculo ou filtragem

---

## Próximas Melhorias (Futuros)

- [ ] Reportar qual ATM foi rejeitado por JORNADA vazia no relatório
- [ ] Estatísticas de rejeição por motivo (Sáb/Dom vs JORNADA vazia)
- [ ] UI para visualizar linhas rejeitadas ao expandir funcionário

