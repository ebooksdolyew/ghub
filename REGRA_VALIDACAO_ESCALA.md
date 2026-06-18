# 📋 REGRA DE VALIDAÇÃO — ATESTADOS POR ESCALA

## Objetivo

Garantir que a contagem de atestados médicos (ATM) respeite a escala de trabalho do funcionário, desconsiderando ATMs registrados em dias que não são úteis para a jornada contratada.

---

## Regra Implementada

### Escala Convencional (Segunda a Sexta)

**ATMs Válidos:** ✅ Seg, Ter, Qua, Qui, Sex  
**ATMs Rejeitados:** ❌ Sab, Dom

Um atestado médico registrado em sábado ou domingo **não é contabilizado** na quantidade final, pois o funcionário não trabalha nesses dias.

**Exemplo:**
```
Silvana Arruda da Silva (Convencional)

ATM detectados (antes validação): 6 dias
├─ 04-Qua (04/02) → ✅ VÁLIDO (quarta = útil)
├─ 05-Qui (05/02) → ✅ VÁLIDO (quinta = útil)
├─ 06-Sex (06/02) → ✅ VÁLIDO (sexta = útil)
├─ 11-Qua (11/02) → ✅ VÁLIDO (quarta = útil)
├─ 12-Qui (12/02) → ✅ VÁLIDO (quinta = útil)
└─ 13-Sex (13/02) → ✅ VÁLIDO (sexta = útil)

ATM válidos (após validação): 6 dias
│
├─ Se 02-Sab ou 03-Dom existisse:
│  └─ Seria rejeitado (não são dias úteis)
│
```

### Escala 12×36 (Turno)

**ATMs Válidos:** ✅ Seg, Ter, Qua, Qui, Sex, Sab, Dom

Nenhuma rejeição. Funcionários em 12×36 podem trabalhar qualquer dia da semana, então todos os ATMs são contabilizados.

**Exemplo:**
```
João da Silva (12×36)

ATM detectados: 6 dias
├─ 04-Qua → ✅ VÁLIDO
├─ 05-Qui → ✅ VÁLIDO
├─ 06-Sex → ✅ VÁLIDO
├─ 11-Qua → ✅ VÁLIDO
├─ 12-Qui → ✅ VÁLIDO
└─ 13-Sex → ✅ VÁLIDO
(+ Sab e Dom se existissem)

ATM válidos (após validação): 6 dias
(nenhuma rejeição para 12×36)
```

---

## Implementação no Código

### Função Principal: `validateAtmsByScale()`

```javascript
function validateAtmsByScale(atmDays, is12) {
  const validos = [];
  const rejeitados = [];
  
  for (const atm of atmDays) {
    // Escala 12×36: aceita todos
    if (is12) {
      validos.push(atm);
      continue;
    }
    
    // Escala convencional: extrai dia da semana
    const dowMatch = atm.display.match(/\((\w+)\)$/);
    if (!dowMatch) {
      validos.push(atm); // sem info, mantém por segurança
      continue;
    }
    
    const dow = dowMatch[1].toUpperCase();
    
    // REJEITA sábado e domingo
    if (dow === 'SAB' || dow === 'DOM') {
      rejeitados.push({ atm, motivo: 'Fim de semana' });
      continue;
    }
    
    // VÁLIDO: seg-sex
    validos.push(atm);
  }
  
  return { validos, rejeitados };
}
```

### Integração em `processPage()`

```javascript
function processPage(pg) {
  // ... extrai nome, escala, período ...
  
  // Lê ATM do PDF
  const atmDetectados = detectAtmByRow(...);
  
  // ← APLICA VALIDAÇÃO AQUI
  const { validos: atmDays, rejeitados: atmRejeitados } = 
    validateAtmsByScale(atmDetectados, is12);
  
  // atmDays agora contém apenas ATMs válidos para a escala
  // atmRejeitados rastreia o que foi descartado
  
  return {
    atmDays,        // ← dados validados
    atm: atmDays.length,  // ← contagem final (após validação)
    atmDetectados,  // ← dados brutos (para auditoria)
    atmRejeitados,  // ← rejeitados (para relatório)
    detectionReport // ← inclui estatísticas de rejeição
  };
}
```

---

## Fluxo de Processamento

```
PDF enviado
    ↓
┌─────────────────────────────────────┐
│ 1. Extração de ATM (motor coluna)  │
│    Resultado: 6 ATM detectados      │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ 2. Validação por Escala             │
│                                     │
│  Se Convencional:                   │
│   └─ Rejeita Sáb/Dom                │
│   └─ Mantém Seg-Sex                 │
│                                     │
│  Se 12×36:                          │
│   └─ Mantém todos                   │
│                                     │
│  Resultado: X ATMs válidos          │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ 3. Consolidação Final               │
│    e.atm = contador válido          │
│    CSV exporta apenas válidos       │
│    Relatório mostra rejeições       │
└─────────────────────────────────────┘
```

---

## Saída do Relatório

### Antes (v2.0)
```
Silvana Arruda da Silva
  Escala: Convencional
  Dias ATM detectados: 6
  Datas: 4, 5, 6, 11, 12, 13
```

### Depois (v2.1)
```
Silvana Arruda da Silva
  Escala: Convencional
  ATM detectados: 6         ← bruto
  ATM válidos: 6            ← após validação
  Datas válidas: 4, 5, 6, 11, 12, 13

[Se houvesse Sáb/Dom:]
  ATM rejeitados: 2
    ❌ 02/02/2024 (Sab) — Fim de semana (escala convencional)
    ❌ 03/02/2024 (Dom) — Fim de semana (escala convencional)
```

### No CSV
```
Nome;Escala;ATM;Datas
Silvana Arruda da Silva;Convencional;6;04/02/2024, 05/02/2024, 06/02/2024, 11/02/2024, 12/02/2024, 13/02/2024
```

**Apenas ATMs válidos aparecem.**

---

## Casos Extremos

### Caso 1: Funcionário sem nenhum ATM em dias úteis

```
João (Convencional) tem ATM apenas em:
  02-Sab → ❌ rejeita
  03-Dom → ❌ rejeita

Resultado:
  ATM detectados: 2
  ATM válidos: 0
  Status na tabela: "—" (sem ATM)
```

### Caso 2: ATM em feriado (segunda-feira feriada)

```
Situação: 02-Seg é feriado prolongado
ATM registrado no PDF: 02-Seg

Decisão: MANTÉM (segunda = dia útil por escala)
Motivo: Sistema não detecta feriados, apenas dias da semana

Nota: Se seu RH necessita descartar ATM em feriados específicos,
      isso seria uma melhoria futura (v2.2+)
```

### Caso 3: Funcionário 12×36 com ATM em fim de semana

```
Carlos (12×36) tem ATM:
  04-Qua → ✅ válido
  05-Qui → ✅ válido
  13-Sab → ✅ válido (12×36 trabalha sábados)

Resultado:
  ATM válidos: 3 (nenhuma rejeição)
```

---

## Garantias de Auditoria

1. **Rastreabilidade:** Cada ATM rejeitado é registrado com motivo
2. **Transparência:** Relatório mostra antes/depois da validação
3. **Reversibilidade:** Dados brutos mantidos para reprocessamento
4. **Integridade:** CSV e PDF exports refletem dados validados

---

## Testes Recomendados

Crie um PDF de teste com:

```
Funcionário: TESTE_CONVENCIONAL
01-Seg: ATM ✅ Será contado
02-Sab: ATM ❌ Será rejeitado
03-Dom: ATM ❌ Será rejeitado
05-Ter: ATM ✅ Será contado

Esperado no CSV: 2 ATMs
Relatório deve mostrar: "2 rejeitados (Sab/Dom)"
```

---

## Próximas Melhorias (v2.2+)

- [ ] Detecção automática de feriados
- [ ] Opção de "ignorar ATMs em feriados prolongados"
- [ ] UI mostrando motivo da rejeição ao expandir funcionário
- [ ] Histórico de rejeições por funcionário

---

## Suporte

Se a contagem não corresponder ao esperado:

1. Abra o **Relatório** (botão 📈)
2. Procure pela seção "VALIDAÇÃO POR ESCALA"
3. Verifique:
   - `ATM detectados` (antes da validação)
   - `ATM rejeitados` (rejeições por fim de semana)
   - `ATM válidos` (contagem final usada)

4. Envie screenshot do relatório para debug

