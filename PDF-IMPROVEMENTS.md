# PDF Improvements — Cook Planner

## Problemas atuais

### 1. Checkbox "☐" renderiza como "&"
A fonte Helvetica embutida no jsPDF não suporta o caractere Unicode ☐. Precisa desenhar o checkbox manualmente com `doc.rect()`.

### 2. Tabela do cardápio espremida
As colunas somam mais que a largura útil da página. Precisa ajustar proporções.

### 3. Design genérico
Os PDFs não refletem o tema Botanical Garden do app. Precisam ter a mesma identidade visual.

---

## Solução: Checkboxes desenhados

Substituir TODAS as ocorrências de `doc.text("☐ " + ...)` por checkboxes desenhados com `rect()`:

```javascript
// Helper function para desenhar checkbox + texto
function drawCheckbox(doc, x, y, text, fontSize = 8.5, textColor = [44, 44, 44]) {
  const boxSize = 3.2;
  const boxY = y - boxSize + 0.5;
  
  // Desenha o quadrado do checkbox
  doc.setDrawColor(138, 133, 120);
  doc.setLineWidth(0.3);
  doc.rect(x, boxY, boxSize, boxSize);
  
  // Texto ao lado
  doc.setFont("helvetica", "normal");
  doc.setFontSize(fontSize);
  doc.setTextColor(...textColor);
  doc.text(text, x + boxSize + 2.5, y);
}
```

Usar em vez de `doc.text("☐ " + item, ...)`:
```javascript
drawCheckbox(doc, marginL + 4, y, item);
```

---

## Solução: PDF do Cardápio da Semana (redesign)

```javascript
function downloadPDFMenu() {
  const { jsPDF } = window.jspdf;
  const doc = new jsPDF({ orientation: "portrait", unit: "mm", format: "a4" });
  const dateRange = getWeekDateRange();
  const pageW = 210;
  const marginL = 15;
  const marginR = 15;
  const usableW = pageW - marginL - marginR;

  // ── Decorative header bar ──
  doc.setFillColor(74, 124, 89); // fern-green
  doc.rect(0, 0, pageW, 3, "F");

  // ── Title ──
  doc.setFont("helvetica", "bold");
  doc.setFontSize(20);
  doc.setTextColor(74, 124, 89);
  doc.text("Cardápio da Semana", pageW / 2, 18, { align: "center" });

  // ── Subtitle with decorative line ──
  doc.setFont("helvetica", "normal");
  doc.setFontSize(10);
  doc.setTextColor(138, 133, 120);
  doc.text(dateRange, pageW / 2, 26, { align: "center" });

  // Decorative line under subtitle
  doc.setDrawColor(232, 237, 229);
  doc.setLineWidth(0.5);
  doc.line(marginL + 20, 30, pageW - marginR - 20, 30);

  // ── Table ──
  const tableBody = CONFIG.days.map(day => {
    const dayState = appState.weekPlan[day];
    const formatMeal = (slots) => {
      if (!slots) return "—";
      const parts = [slots.protein, slots.side, slots.veggie].filter(Boolean);
      return parts.length ? parts.join("\n+ ") : "—";
    };
    const lunch = formatMeal(dayState.lunch);
    let dinner;
    if (!dayState.dinnerExpanded || !dayState.dinner) {
      dinner = "Sobra do almoço";
    } else {
      dinner = formatMeal(dayState.dinner);
    }
    // Use short day name
    const shortDay = day.replace("-feira", "");
    return [shortDay, lunch, dinner];
  });

  doc.autoTable({
    head: [["Dia", "Almoço", "Jantar"]],
    body: tableBody,
    startY: 36,
    styles: {
      font: "helvetica",
      fontSize: 9,
      cellPadding: { top: 4, right: 4, bottom: 4, left: 4 },
      overflow: "linebreak",
      lineColor: [232, 237, 229],
      lineWidth: 0.3,
      valign: "middle"
    },
    headStyles: {
      fillColor: [74, 124, 89],
      textColor: [255, 255, 255],
      fontStyle: "bold",
      fontSize: 9,
      halign: "center"
    },
    alternateRowStyles: {
      fillColor: [245, 243, 237]
    },
    bodyStyles: {
      textColor: [44, 44, 44]
    },
    columnStyles: {
      0: { cellWidth: 28, fontStyle: "bold", halign: "center", fillColor: [232, 237, 229] },
      1: { cellWidth: (usableW - 28) / 2 },
      2: { cellWidth: (usableW - 28) / 2 }
    },
    margin: { left: marginL, right: marginR },
    tableLineColor: [232, 237, 229],
    tableLineWidth: 0.3
  });

  // ── Footer decorative bar ──
  doc.setFillColor(74, 124, 89);
  doc.rect(0, 294, pageW, 3, "F");

  doc.save(`cardapio-semana-${dateRange.replace(/[^0-9a-zA-Z]/g, "-")}.pdf`);
}
```

**Mudanças principais:**
- Barra decorativa verde no topo e rodapé
- Linha decorativa sob o subtítulo
- Dia sem "-feira" (Segunda em vez de Segunda-feira) pra caber melhor
- Pratos em linhas separadas com "\n+ " entre eles pra melhor leitura
- Coluna "Dia" com fundo sage e centralizada
- Largura das colunas calculadas proporcionalmente à largura útil

---

## Solução: PDF da Lista de Compras (redesign)

Aplicar as mesmas melhorias visuais e usar `drawCheckbox()` em vez de `☐`:

```javascript
function downloadPDFShopping() {
  const { jsPDF } = window.jspdf;
  const doc = new jsPDF({ orientation: "portrait", unit: "mm", format: "a4" });
  const dateRange = getWeekDateRange();
  const pageW = 210;
  const marginL = 15;
  const marginR = 15;
  const usableW = pageW - marginL - marginR;
  const pageH = 280;
  let y = 0;

  // ── Decorative header bar ──
  doc.setFillColor(74, 124, 89);
  doc.rect(0, 0, pageW, 3, "F");

  // ── Title ──
  doc.setFont("helvetica", "bold");
  doc.setFontSize(20);
  doc.setTextColor(74, 124, 89);
  doc.text("Lista de Compras", pageW / 2, 18, { align: "center" });

  doc.setFont("helvetica", "normal");
  doc.setFontSize(10);
  doc.setTextColor(138, 133, 120);
  doc.text(`Semana ${dateRange}`, pageW / 2, 26, { align: "center" });

  doc.setDrawColor(232, 237, 229);
  doc.setLineWidth(0.5);
  doc.line(marginL + 20, 30, pageW - marginR - 20, 30);

  y = 38;

  // ── Section header helper ──
  const addSectionHeader = (title, bgColor) => {
    if (y > pageH - 20) { doc.addPage(); y = 20; }
    doc.setFillColor(...bgColor);
    doc.roundedRect(marginL, y, usableW, 8, 2, 2, "F");
    doc.setFont("helvetica", "bold");
    doc.setFontSize(9);
    doc.setTextColor(255, 255, 255);
    doc.text(title, marginL + 4, y + 5.5);
    y += 12;
  };

  // ── Pantry section ──
  addSectionHeader("CONFERE SE TEM EM CASA", [249, 166, 32]); // marigold
  CONFIG.pantryStaples.forEach(item => {
    if (y > pageH - 10) { doc.addPage(); y = 20; }
    drawCheckbox(doc, marginL + 4, y, item);
    y += 6;
  });
  y += 6;

  // ── Shopping items by store ──
  const shoppingData = buildShoppingData();
  if (Object.keys(shoppingData).length === 0) {
    doc.setFont("helvetica", "italic");
    doc.setFontSize(9);
    doc.setTextColor(138, 133, 120);
    doc.text("Nenhum ingrediente no cardápio.", marginL + 4, y);
  } else {
    Object.entries(shoppingData).forEach(([store, items]) => {
      addSectionHeader(store.toUpperCase(), [74, 124, 89]); // fern-green
      items.forEach(entry => {
        if (y > pageH - 14) { doc.addPage(); y = 20; }
        // Item name with checkbox
        drawCheckbox(doc, marginL + 4, y, entry.item, 9, [44, 44, 44]);
        y += 4.5;
        // Context in smaller gray text, indented
        const contextStr = entry.contexts.map(c => c.label).join(", ");
        doc.setFont("helvetica", "italic");
        doc.setFontSize(7.5);
        doc.setTextColor(138, 133, 120);
        doc.text(contextStr, marginL + 10, y);
        y += 6;
      });
      y += 3;
    });
  }

  // ── Footer decorative bar ──
  doc.setFillColor(74, 124, 89);
  doc.rect(0, 294, pageW, 3, "F");

  doc.save(`lista-compras-${dateRange.replace(/[^0-9a-zA-Z]/g, "-")}.pdf`);
}
```

---

## Resumo das alterações

1. **Criar** a função `drawCheckbox()` como helper global
2. **Substituir** `downloadPDFMenu()` inteira pelo código acima
3. **Substituir** `downloadPDFShopping()` inteira pelo código acima
4. **Remover** todas as ocorrências de `"☐ "` em strings de texto do PDF

## Resultado esperado

- Checkboxes visuais limpos (quadradinhos desenhados) em vez de "&"
- Barra verde decorativa no topo e rodapé de ambos os PDFs
- Linha decorativa abaixo do subtítulo
- Tabela do cardápio com proporções corretas e dias abreviados
- Pratos listados com quebra de linha entre componentes
- Identidade visual Botanical Garden consistente com o app
