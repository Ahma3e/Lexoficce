# 🔧 Troubleshooting & Problemlösungen
## Lexware-Zapier Automatisierung

---

## 🚨 Häufige Probleme und Lösungen

### Problem 1: API-Verbindungsfehler zu Lexware

#### **Symptome:**
- Zapier-Workflow bricht mit "401 Unauthorized" ab
- Keine Daten werden von Lexware abgerufen
- Fehler: "API-Token ungültig"

#### **Diagnoseschritte:**
```javascript
// Schritt 1: API-Token prüfen
const testLexwareConnection = async () => {
  try {
    const response = await fetch('https://api.lexware.de/v2/auth/test', {
      headers: {
        'Authorization': 'Bearer YOUR_TOKEN',
        'Content-Type': 'application/json'
      }
    });
    
    if (response.status === 401) {
      console.log("❌ Token ungültig oder abgelaufen");
      return false;
    } else if (response.status === 200) {
      console.log("✅ API-Verbindung erfolgreich");
      return true;
    }
  } catch (error) {
    console.log("❌ Netzwerkfehler:", error.message);
    return false;
  }
};

// Schritt 2: Token-Berechtigung prüfen
const checkPermissions = async () => {
  const requiredPermissions = [
    'invoices:read',
    'customers:read',
    'documents:read'
  ];
  
  // API-Call zu Permissions-Endpoint
  // Implementation spezifisch für Lexware API
};
```

#### **Lösungsansätze:**
1. **Token erneuern:**
   - In Lexware → Einstellungen → API-Zugang
   - Neuen Token generieren
   - In Zapier → Apps → Lexware → Verbindung bearbeiten

2. **Berechtigungen erweitern:**
   ```json
   {
     "required_scopes": [
       "invoices:read",
       "customers:read", 
       "tax_rates:read",
       "categories:read"
     ]
   }
   ```

3. **Rate-Limiting beachten:**
   ```javascript
   // Zapier Code: Rate-Limiting implementieren
   const delay = (ms) => new Promise(resolve => setTimeout(resolve, ms));
   
   const apiCallWithRetry = async (url, options, maxRetries = 3) => {
     for (let i = 0; i < maxRetries; i++) {
       try {
         const response = await fetch(url, options);
         
         if (response.status === 429) {
           // Rate-Limit erreicht, warten
           await delay(Math.pow(2, i) * 1000);
           continue;
         }
         
         return response;
       } catch (error) {
         if (i === maxRetries - 1) throw error;
         await delay(1000);
       }
     }
   };
   ```

---

### Problem 2: Unvollständige oder fehlerhafte Datenextraktion

#### **Symptome:**
- Fehlende Rechnungen in der Ausgabe
- Falsche Beträge oder Daten
- Rechnungen werden doppelt exportiert

#### **Diagnosecode:**
```javascript
// Datenqualitäts-Check
const validateDataCompleteness = (rawData, expectedCount) => {
  const issues = [];
  
  // Vollständigkeits-Check
  if (rawData.length !== expectedCount) {
    issues.push({
      type: 'count_mismatch',
      expected: expectedCount,
      actual: rawData.length,
      severity: 'high'
    });
  }
  
  // Duplikats-Check
  const invoiceNumbers = rawData.map(r => r.invoice_number);
  const duplicates = invoiceNumbers.filter((num, index) => 
    invoiceNumbers.indexOf(num) !== index
  );
  
  if (duplicates.length > 0) {
    issues.push({
      type: 'duplicates',
      duplicates: duplicates,
      severity: 'medium'
    });
  }
  
  // Datenintegrität-Check
  rawData.forEach((invoice, index) => {
    if (!invoice.invoice_number) {
      issues.push({
        type: 'missing_invoice_number',
        index: index,
        severity: 'high'
      });
    }
    
    if (!invoice.service_date || isNaN(Date.parse(invoice.service_date))) {
      issues.push({
        type: 'invalid_service_date',
        index: index,
        invoice_number: invoice.invoice_number,
        severity: 'high'
      });
    }
    
    if (!invoice.amount || isNaN(parseFloat(invoice.amount))) {
      issues.push({
        type: 'invalid_amount',
        index: index,
        invoice_number: invoice.invoice_number,
        severity: 'high'
      });
    }
  });
  
  return issues;
};
```

#### **Lösungsstrategien:**
1. **Paginierung implementieren:**
   ```javascript
   const getAllInvoices = async (startDate, endDate) => {
     let allInvoices = [];
     let page = 1;
     let hasMore = true;
     
     while (hasMore) {
       const response = await fetch(`https://api.lexware.de/v2/invoices`, {
         method: 'POST',
         headers: {
           'Authorization': 'Bearer ' + apiToken,
           'Content-Type': 'application/json'
         },
         body: JSON.stringify({
           dateRange: { from: startDate, to: endDate },
           page: page,
           limit: 100
         })
       });
       
       const data = await response.json();
       allInvoices = allInvoices.concat(data.invoices);
       
       hasMore = data.hasMore;
       page++;
       
       // Rate-Limiting respektieren
       await delay(200);
     }
     
     return allInvoices;
   };
   ```

2. **Delta-Synchronisation:**
   ```javascript
   const getDeltaInvoices = async (lastSyncTimestamp) => {
     return await fetch(`https://api.lexware.de/v2/invoices/delta`, {
       method: 'POST',
       headers: {
         'Authorization': 'Bearer ' + apiToken,
         'Content-Type': 'application/json'
       },
       body: JSON.stringify({
         since: lastSyncTimestamp,
         includeDeleted: true
       })
     });
   };
   ```

---

### Problem 3: CSV-Formatierungsprobleme

#### **Symptome:**
- Umlaute werden falsch dargestellt
- Kommas in Beschreibungen zerstören CSV-Struktur
- Excel kann CSV nicht korrekt öffnen

#### **Robuste CSV-Generierung:**
```javascript
const generateRobustCSV = (data, format = 'lexware') => {
  const configs = {
    lexware: {
      delimiter: ',',
      encoding: 'UTF-8-BOM',
      dateFormat: 'de-DE',
      decimalSeparator: ','
    },
    datev: {
      delimiter: ';',
      encoding: 'ANSI',
      dateFormat: 'DATEV',
      decimalSeparator: ','
    },
    excel: {
      delimiter: ',',
      encoding: 'UTF-8-BOM',
      dateFormat: 'ISO',
      decimalSeparator: '.'
    }
  };
  
  const config = configs[format];
  
  // CSV-Header mit BOM für Excel-Kompatibilität
  let csvContent = config.encoding === 'UTF-8-BOM' ? '\uFEFF' : '';
  
  // Header-Zeile
  const headers = getHeadersForFormat(format);
  csvContent += headers.map(h => escapeCSVField(h, config.delimiter)).join(config.delimiter) + '\n';
  
  // Datenzeilen
  data.forEach(row => {
    const csvRow = headers.map(header => {
      const value = getFieldValue(row, header, config);
      return escapeCSVField(value, config.delimiter);
    });
    
    csvContent += csvRow.join(config.delimiter) + '\n';
  });
  
  return {
    content: csvContent,
    filename: generateFilename(format),
    mimeType: 'text/csv;charset=utf-8'
  };
};

// Sichere CSV-Feld-Escape-Funktion
const escapeCSVField = (field, delimiter) => {
  if (field === null || field === undefined) return '';
  
  const stringField = String(field);
  
  // Wenn das Feld Delimiter, Anführungszeichen oder Zeilenwechsel enthält
  if (stringField.includes(delimiter) || 
      stringField.includes('"') || 
      stringField.includes('\n') || 
      stringField.includes('\r')) {
    
    // Anführungszeichen doppeln und ganzes Feld in Anführungszeichen setzen
    return '"' + stringField.replace(/"/g, '""') + '"';
  }
  
  return stringField;
};

// Formatspezifische Feldwerte
const getFieldValue = (row, header, config) => {
  const value = row[header];
  
  // Datums-Formatierung
  if (header.includes('datum') && value) {
    const date = new Date(value);
    
    switch(config.dateFormat) {
      case 'de-DE':
        return date.toLocaleDateString('de-DE');
      case 'DATEV':
        return date.toLocaleDateString('de-DE').replace(/\./g, '');
      case 'ISO':
        return date.toISOString().split('T')[0];
      default:
        return value;
    }
  }
  
  // Zahlen-Formatierung
  if (header.includes('betrag') && value) {
    const number = parseFloat(value);
    return config.decimalSeparator === ',' 
      ? number.toFixed(2).replace('.', ',')
      : number.toFixed(2);
  }
  
  return value || '';
};
```

---

### Problem 4: Google Drive Upload-Fehler

#### **Symptome:**
- "Quota exceeded" Fehler
- Upload-Timeouts
- Berechtigungsfehler

#### **Robuste Upload-Lösung:**
```javascript
const uploadToGoogleDrive = async (fileContent, filename, folderId) => {
  const maxRetries = 3;
  const chunkSize = 5 * 1024 * 1024; // 5MB Chunks
  
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      // Kleiner Dateien: Direkter Upload
      if (fileContent.length < chunkSize) {
        return await directUpload(fileContent, filename, folderId);
      }
      
      // Große Dateien: Chunked Upload
      return await chunkedUpload(fileContent, filename, folderId, chunkSize);
      
    } catch (error) {
      console.log(`Upload-Versuch ${attempt} fehlgeschlagen:`, error.message);
      
      if (attempt === maxRetries) {
        throw new Error(`Upload nach ${maxRetries} Versuchen fehlgeschlagen: ${error.message}`);
      }
      
      // Exponential Backoff
      await delay(Math.pow(2, attempt) * 1000);
    }
  }
};

const chunkedUpload = async (fileContent, filename, folderId, chunkSize) => {
  // Resumable Upload Session starten
  const sessionUrl = await initResumableUpload(filename, folderId);
  
  const totalSize = fileContent.length;
  let uploadedBytes = 0;
  
  while (uploadedBytes < totalSize) {
    const chunkEnd = Math.min(uploadedBytes + chunkSize, totalSize);
    const chunk = fileContent.slice(uploadedBytes, chunkEnd);
    
    const response = await uploadChunk(sessionUrl, chunk, uploadedBytes, totalSize);
    
    if (response.status === 200 || response.status === 201) {
      // Upload komplett
      return await response.json();
    } else if (response.status === 308) {
      // Weiter uploaden
      uploadedBytes = getUploadedBytes(response);
    } else {
      throw new Error(`Chunk-Upload fehlgeschlagen: ${response.status}`);
    }
  }
};
```

---

### Problem 5: E-Mail-Versand-Probleme

#### **Symptome:**
- E-Mails kommen nicht an
- Anhänge sind zu groß
- Spam-Filter blockiert E-Mails

#### **Optimierte E-Mail-Lösung:**
```javascript
const sendIntelligentEmail = async (emailData) => {
  // Anhang-Größe prüfen
  const maxAttachmentSize = 25 * 1024 * 1024; // 25MB Gmail-Limit
  let attachments = emailData.attachments || [];
  
  // Große Anhänge durch Links ersetzen
  const processedAttachments = await Promise.all(
    attachments.map(async (attachment) => {
      if (attachment.size > maxAttachmentSize) {
        // Upload zu Google Drive und Link senden
        const driveFile = await uploadToGoogleDrive(
          attachment.content, 
          attachment.filename,
          'email_attachments_folder'
        );
        
        return {
          type: 'link',
          filename: attachment.filename,
          url: driveFile.webViewLink,
          note: 'Datei zu groß für E-Mail-Anhang, verfügbar unter:'
        };
      }
      
      return attachment;
    })
  );
  
  // E-Mail-Template anpassen
  const emailTemplate = {
    to: emailData.recipients,
    subject: emailData.subject,
    html: await generateEmailHTML(emailData, processedAttachments),
    attachments: processedAttachments.filter(a => a.type !== 'link')
  };
  
  // Anti-Spam Optimierungen
  emailTemplate.headers = {
    'X-Priority': '3',
    'X-MSMail-Priority': 'Normal',
    'X-Mailer': 'Lexware Automation System',
    'X-Auto-Response-Suppress': 'DR, RN, NRN, OOF, AutoReply'
  };
  
  return await sendEmail(emailTemplate);
};

const generateEmailHTML = async (emailData, attachments) => {
  const linkAttachments = attachments.filter(a => a.type === 'link');
  
  return `
    <!DOCTYPE html>
    <html>
    <head>
      <meta charset="UTF-8">
      <style>
        body { font-family: Arial, sans-serif; line-height: 1.6; }
        .header { background: #f4f4f4; padding: 20px; text-align: center; }
        .content { padding: 20px; }
        .attachments { background: #e8f4f8; padding: 15px; margin: 20px 0; }
        .link-attachment { 
          display: block; 
          margin: 10px 0; 
          padding: 10px; 
          background: white; 
          border-left: 4px solid #007cba; 
        }
      </style>
    </head>
    <body>
      <div class="header">
        <h2>📊 Lexware Rechnungsauswertung</h2>
        <p>Automatisch generiert am ${new Date().toLocaleDateString('de-DE')}</p>
      </div>
      
      <div class="content">
        ${emailData.content}
        
        ${linkAttachments.length > 0 ? `
          <div class="attachments">
            <h3>📎 Verfügbare Dateien:</h3>
            ${linkAttachments.map(att => `
              <div class="link-attachment">
                <strong>${att.filename}</strong><br>
                <small>${att.note}</small><br>
                <a href="${att.url}" target="_blank">📥 Datei öffnen</a>
              </div>
            `).join('')}
          </div>
        ` : ''}
      </div>
      
      <div style="text-align: center; color: #666; font-size: 12px; margin-top: 30px;">
        <p>Diese E-Mail wurde automatisch generiert von Ihrem Lexware Automatisierungssystem.</p>
      </div>
    </body>
    </html>
  `;
};
```

---

## 🔧 Performance-Optimierung

### Datenbankabfrage-Optimierung
```javascript
// Intelligente Abfrage-Strategien
const optimizedDataRetrieval = {
  // Inkrementelle Synchronisation
  incrementalSync: async (lastSyncTimestamp) => {
    const query = {
      modifiedSince: lastSyncTimestamp,
      includedeleted: true,
      fields: ['id', 'invoice_number', 'service_date', 'amount', 'status', 'modified_date']
    };
    
    return await lexwareAPI.getInvoices(query);
  },
  
  // Batch-Verarbeitung
  batchProcessing: async (invoiceIds, batchSize = 50) => {
    const results = [];
    
    for (let i = 0; i < invoiceIds.length; i += batchSize) {
      const batch = invoiceIds.slice(i, i + batchSize);
      const batchResults = await lexwareAPI.getInvoiceDetails(batch);
      results.push(...batchResults);
      
      // Rate-Limiting
      await delay(100);
    }
    
    return results;
  },
  
  // Caching-Strategie
  cacheStrategy: {
    customerData: {
      ttl: 24 * 60 * 60 * 1000, // 24 Stunden
      key: 'lexware_customers'
    },
    taxRates: {
      ttl: 7 * 24 * 60 * 60 * 1000, // 7 Tage
      key: 'lexware_tax_rates'
    }
  }
};
```

### Memory-Management
```javascript
const memoryOptimizedProcessing = (largeDataset) => {
  const CHUNK_SIZE = 1000;
  const processedResults = [];
  
  // Stream-Processing für große Datenmengen
  for (let i = 0; i < largeDataset.length; i += CHUNK_SIZE) {
    const chunk = largeDataset.slice(i, i + CHUNK_SIZE);
    
    // Chunk verarbeiten
    const processedChunk = processInvoiceChunk(chunk);
    processedResults.push(...processedChunk);
    
    // Memory cleanup zwischen Chunks
    if (i % (CHUNK_SIZE * 10) === 0) {
      // Garbage Collection hint
      if (global.gc) global.gc();
    }
  }
  
  return processedResults;
};
```

---

## 📊 Monitoring und Alerting

### Custom Monitoring Dashboard
```javascript
const monitoringMetrics = {
  systemHealth: {
    apiResponseTime: [],
    errorRate: 0,
    successfulExecutions: 0,
    failedExecutions: 0,
    lastSuccessfulRun: null
  },
  
  dataQuality: {
    recordsProcessed: 0,
    validRecords: 0,
    invalidRecords: 0,
    duplicatesFound: 0,
    dataCompletenessScore: 100
  },
  
  businessMetrics: {
    totalRevenue: 0,
    invoiceCount: 0,
    averageProcessingTime: 0,
    customerCount: 0
  }
};

const updateMetrics = (executionData) => {
  // System Health
  monitoringMetrics.systemHealth.apiResponseTime.push(executionData.apiResponseTime);
  monitoringMetrics.systemHealth.lastSuccessfulRun = new Date();
  
  if (executionData.success) {
    monitoringMetrics.systemHealth.successfulExecutions++;
  } else {
    monitoringMetrics.systemHealth.failedExecutions++;
  }
  
  // Error Rate berechnen
  const totalExecutions = monitoringMetrics.systemHealth.successfulExecutions + 
                         monitoringMetrics.systemHealth.failedExecutions;
  monitoringMetrics.systemHealth.errorRate = 
    (monitoringMetrics.systemHealth.failedExecutions / totalExecutions) * 100;
  
  // Data Quality
  monitoringMetrics.dataQuality.recordsProcessed += executionData.recordsProcessed;
  monitoringMetrics.dataQuality.validRecords += executionData.validRecords;
  monitoringMetrics.dataQuality.invalidRecords += executionData.invalidRecords;
  
  // Business Metrics
  monitoringMetrics.businessMetrics.totalRevenue += executionData.revenue;
  monitoringMetrics.businessMetrics.invoiceCount += executionData.invoiceCount;
};
```

### Intelligente Alert-Rules
```javascript
const alertRules = {
  critical: [
    {
      name: 'API_DOWN',
      condition: () => monitoringMetrics.systemHealth.errorRate > 50,
      message: '🚨 KRITISCH: API-Ausfallrate über 50%',
      action: 'immediate_notification'
    },
    {
      name: 'DATA_LOSS',
      condition: () => monitoringMetrics.dataQuality.dataCompletenessScore < 90,
      message: '🚨 KRITISCH: Datenverlust erkannt',
      action: 'emergency_backup'
    }
  ],
  
  warning: [
    {
      name: 'HIGH_ERROR_RATE',
      condition: () => monitoringMetrics.systemHealth.errorRate > 10,
      message: '⚠️ WARNUNG: Erhöhte Fehlerrate',
      action: 'investigation_needed'
    },
    {
      name: 'SLOW_PROCESSING',
      condition: () => {
        const avgResponseTime = monitoringMetrics.systemHealth.apiResponseTime
          .slice(-10).reduce((a, b) => a + b, 0) / 10;
        return avgResponseTime > 30000; // 30 Sekunden
      },
      message: '⚠️ WARNUNG: Langsame Verarbeitung',
      action: 'performance_check'
    }
  ]
};
```

---

Diese erweiterte Troubleshooting-Anleitung hilft Ihnen bei der Lösung der häufigsten Probleme und stellt sicher, dass Ihre Automatisierung stabil und zuverlässig läuft.
