# Mise en œuvre de BIRT (Business Intelligence and Reporting Tools) avec Spring Boot

## 1. configuration minimale

### 1.1 Structure de projet recommande 
```
src/
├── main/
│   ├── java/
│   │   └── com/example/birtdemo/
│   │       ├── config/
│   │       ├── controller/
│   │       ├── model/
│   │       ├── service/
│   │       └── BirtDemoApplication.java
│   └── resources/
│       ├── reports/          # Dossier pour les fichiers .rptdesign
│       ├── static/           # Fichiers statiques
│       └── templates/        # Fichiers de template
```

### 1.2 Ajouter les dependances Maven

```xml
<dependencies>
    <!-- Spring Boot Starter Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- BIRT Runtime -->
    <dependency>
        <groupId>org.eclipse.birt.runtime</groupId>
        <artifactId>org.eclipse.birt.runtime</artifactId>
        <version>4.8.0</version>
    </dependency>

    <!-- Pour l'export en Excel -->
    <dependency>
        <groupId>org.apache.poi</groupId>
        <artifactId>poi</artifactId>
        <version>5.2.2</version>
    </dependency>
</dependencies>

```
## 2. Creation d'un rapport BIRT simple

### 2.1 Créer un fichier rapport (simple_report.rptdesign)

Vous pouvez utiliser BIRT Designer pour créer le rapport ou créer manuellement un fichier XML. Voici un exemple minimal :
```xml
<?xml version="1.0" encoding="UTF-8"?>
<report xmlns="http://www.eclipse.org/birt/2005/design" version="3.2.23" id="1">
    <property name="createdBy">Eclipse BIRT Designer</property>
    <property name="units">in</property>
    
    <data-sources>
        <oda-data-source extensionID="org.eclipse.birt.report.data.oda.jdbc" name="Data Source" id="7">
            <property name="odaDriverClass">org.h2.Driver</property>
            <property name="odaURL">jdbc:h2:mem:testdb</property>
            <property name="odaUser">sa</property>
        </oda-data-source>
    </data-sources>
    
    <data-sets>
        <oda-data-set extensionID="org.eclipse.birt.report.data.oda.jdbc.JdbcSelectDataSet" name="Sample Data Set" id="8">
            <structure>
                <list-property name="columnHints">
                    <structure>
                        <property name="columnName">DUMMY</property>
                        <property name="displayName">Dummy Column</property>
                    </structure>
                </list-property>
            </structure>
            <property name="dataSource">Data Source</property>
            <property name="queryText">SELECT 'Hello BIRT from Spring Boot!' as DUMMY</property>
        </oda-data-set>
    </data-sets>
    
    <body>
        <label id="9">
            <text-property name="text">Hello BIRT!</text-property>
        </label>
        <data id="10">
            <property name="dataSet">Sample Data Set</property>
            <list-property name="boundDataColumns">
                <structure>
                    <property name="name">DUMMY</property>
                    <text-property name="displayName">DUMMY</text-property>
                    <expression name="expression" type="javascript">dataSetRow["DUMMY"]</expression>
                </structure>
            </list-property>
        </data>
    </body>
</report>
```
## 3. Service de generation de rapports BIRT

### 3.1 Creation d'un service BirtReportService
```java
package com.example.birtdemo.service;

import org.eclipse.birt.core.exception.BirtException;
import org.eclipse.birt.core.framework.Platform;
import org.eclipse.birt.report.engine.api.*;
import org.springframework.stereotype.Service;

import javax.servlet.ServletContext;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.IOException;
import java.util.HashMap;
import java.util.logging.Level;
import java.util.logging.Logger;

@Service
public class BirtReportService {

    private IReportEngine birtEngine;
    private final ServletContext servletContext;

    public BirtReportService(ServletContext servletContext) {
        this.servletContext = servletContext;
        try {
            EngineConfig config = new EngineConfig();
            Platform.startup(config);
            IReportEngineFactory factory = (IReportEngineFactory) Platform
                    .createFactoryObject(IReportEngineFactory.EXTENSION_REPORT_ENGINE_FACTORY);
            birtEngine = factory.createReportEngine(config);
            birtEngine.changeLogLevel(Level.WARNING);
        } catch (BirtException e) {
            throw new RuntimeException("Failed to initialize BIRT engine", e);
        }
    }

    public byte[] generateReport(String reportName, String format, 
                               HttpServletRequest request, 
                               HttpServletResponse response) throws BirtException, IOException {
        
        String reportPath = servletContext.getRealPath("/reports/" + reportName + ".rptdesign");
        
        IReportRunnable design = birtEngine.openReportDesign(reportPath);
        IRunAndRenderTask task = birtEngine.createRunAndRenderTask(design);
        
        // Configuration du rendu
        RenderOption options = null;
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
        
        if ("pdf".equalsIgnoreCase(format)) {
            PDFRenderOption pdfOptions = new PDFRenderOption();
            pdfOptions.setOutputFormat("pdf");
            pdfOptions.setOutputStream(outputStream);
            options = pdfOptions;
        } else if ("html".equalsIgnoreCase(format)) {
            HTMLRenderOption htmlOptions = new HTMLRenderOption();
            htmlOptions.setOutputFormat("html");
            htmlOptions.setOutputStream(outputStream);
            htmlOptions.setHtmlRtLFlag(false);
            htmlOptions.setEmbeddable(true);
            options = htmlOptions;
        } else if ("xls".equalsIgnoreCase(format)) {
            EXCELRenderOption excelOptions = new EXCELRenderOption();
            excelOptions.setOutputFormat("xls");
            excelOptions.setOutputStream(outputStream);
            options = excelOptions;
        }
        
        task.setRenderOption(options);
        
        // Paramètres du rapport
        HashMap<String, Object> params = new HashMap<>();
        params.put("param1", "value1");
        task.setParameterValues(params);
        
        // Contexte d'exécution
        task.getAppContext().put(EngineConstants.APPCONTEXT_BIRT_VIEWER_HTTPSERVET_REQUEST, request);
        
        // Exécution du rapport
        task.run();
        task.close();
        
        return outputStream.toByteArray();
    }
    
    public void destroy() {
        if (birtEngine != null) {
            birtEngine.destroy();
        }
        Platform.shutdown();
    }
}
```
## 4. Controller pour exposer l'API

### 4.1 Créer un controller ReportController

```java
package com.example.birtdemo.controller;

import com.example.birtdemo.service.BirtReportService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@RestController
@RequestMapping("/api/reports")
public class ReportController {

    @Autowired
    private BirtReportService birtReportService;

    @GetMapping("/{reportName}/{format}")
    public ResponseEntity<byte[]> generateReport(
            @PathVariable String reportName,
            @PathVariable String format,
            HttpServletRequest request,
            HttpServletResponse response) {
        
        try {
            byte[] reportContent = birtReportService.generateReport(reportName, format, request, response);
            
            HttpHeaders headers = new HttpHeaders();
            headers.setContentType(getMediaType(format));
            headers.setContentDispositionFormData("filename", reportName + "." + format);
            
            return ResponseEntity.ok()
                    .headers(headers)
                    .body(reportContent);
        } catch (Exception e) {
            return ResponseEntity.internalServerError().build();
        }
    }
    
    private MediaType getMediaType(String format) {
        switch (format.toLowerCase()) {
            case "pdf": return MediaType.APPLICATION_PDF;
            case "xls": return MediaType.parseMediaType("application/vnd.ms-excel");
            case "html": return MediaType.TEXT_HTML;
            default: return MediaType.APPLICATION_OCTET_STREAM;
        }
    }
}
```

## 5. Configuration Spring Boot 

### 5.1 Configuration de la plateforme BIRT 
```java
package com.example.birtdemo.config;

import org.eclipse.birt.core.framework.Platform;
import org.eclipse.birt.report.engine.api.EngineConfig;
import org.eclipse.birt.report.engine.api.IReportEngine;
import org.eclipse.birt.report.engine.api.IReportEngineFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.annotation.PreDestroy;
import javax.servlet.ServletContext;
import java.util.logging.Level;

@Configuration
public class BirtConfig {

    private IReportEngine birtEngine;

    @Bean
    public IReportEngine birtEngine(ServletContext servletContext) {
        EngineConfig config = new EngineConfig();
        config.setLogConfig(servletContext.getRealPath("/logs"), Level.WARNING);
        
        try {
            Platform.startup(config);
            IReportEngineFactory factory = (IReportEngineFactory) Platform
                    .createFactoryObject(IReportEngineFactory.EXTENSION_REPORT_ENGINE_FACTORY);
            birtEngine = factory.createReportEngine(config);
            return birtEngine;
        } catch (Exception e) {
            throw new RuntimeException("Failed to initialize BIRT engine", e);
        }
    }

    @PreDestroy
    public void destroy() {
        if (birtEngine != null) {
            birtEngine.destroy();
        }
        Platform.shutdown();
    }
}
```

