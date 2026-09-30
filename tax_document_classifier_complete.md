# Tax Document Classification System

## Overview

This project implements a Node.js + TypeScript workflow for classifying
UK tax PDFs before a tax-comparison process runs.

The system accepts:

-   Current Year PDF (`CY`)
-   Previous Year PDF (`PY`)

Each PDF is analyzed with Azure Document Intelligence, classified using
deterministic rules first, and sent to an AI fallback only when
deterministic classification is ambiguous.

The high-level workflow is:

``` text
CY PDF + PY PDF
      |
      v
Azure Document Intelligence
      |
      v
Page-level text extraction
      |
      v
Deterministic classification
      |
      +---- high confidence ----> FINAL
      |
      +---- ambiguous ----------> AI fallback
                                      |
                                      +---- high confidence --> FINAL
                                      |
                                      +---- low confidence ---> REVIEW_REQUIRED
```

A single PDF can contain both a Tax Return and a Tax Computation.
Therefore, the implementation is designed around document segments/pages
rather than assuming one document type per PDF.

------------------------------------------------------------------------

# 1. Project Structure

``` text
tax-document-classifier/
|
├── src/
│   ├── config.ts
│   ├── types.ts
│   |
│   ├── azure/
│   │   ├── document-intelligence.service.ts
│   │   └── page-extractor.ts
│   |
│   ├── classification/
│   │   ├── rules.ts
│   │   ├── deterministic-classifier.ts
│   │   ├── ai-classifier.ts
│   │   └── hybrid-classifier.ts
│   |
│   ├── services/
│   │   └── document.service.ts
│   |
│   ├── routes/
│   │   └── document.routes.ts
│   |
│   └── server.ts
│
├── uploads/
│
├── .env
├── .gitignore
├── package.json
├── tsconfig.json
└── README.md
```

------------------------------------------------------------------------

# 2. Install Dependencies

Create the project:

``` bash
mkdir tax-document-classifier
cd tax-document-classifier
npm init -y
```

Install runtime dependencies:

``` bash
npm install express multer dotenv openai zod @azure-rest/ai-document-intelligence @azure/core-auth
```

Install development dependencies:

``` bash
npm install -D typescript ts-node-dev @types/node @types/express @types/multer
```

------------------------------------------------------------------------

# 3. package.json

``` json
{
  "name": "tax-document-classifier",
  "version": "1.0.0",
  "main": "dist/server.js",
  "scripts": {
    "dev": "ts-node-dev --respawn --transpile-only src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js"
  },
  "dependencies": {
    "@azure-rest/ai-document-intelligence": "^1.0.0",
    "@azure/core-auth": "^1.9.0",
    "dotenv": "^16.4.5",
    "express": "^5.1.0",
    "multer": "^2.0.2",
    "openai": "^5.0.0",
    "zod": "^3.23.8"
  },
  "devDependencies": {
    "@types/express": "^5.0.0",
    "@types/multer": "^2.0.0",
    "@types/node": "^22.0.0",
    "ts-node-dev": "^2.0.0",
    "typescript": "^5.0.0"
  }
}
```

------------------------------------------------------------------------

# 4. Environment Variables

Create `.env`:

``` env
PORT=3000

AZURE_DOCUMENT_INTELLIGENCE_ENDPOINT=https://YOUR_RESOURCE.cognitiveservices.azure.com
AZURE_DOCUMENT_INTELLIGENCE_KEY=YOUR_AZURE_KEY

OPENAI_API_KEY=YOUR_OPENAI_KEY
OPENAI_MODEL=gpt-5.6-luna

DETERMINISTIC_THRESHOLD=0.90
AI_THRESHOLD=0.85
```

Do not commit `.env` to source control.

------------------------------------------------------------------------

# 5. TypeScript Configuration

Create `tsconfig.json`:

``` json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "CommonJS",
    "moduleResolution": "Node",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": [
    "src/**/*.ts"
  ]
}
```

------------------------------------------------------------------------

# 6. Configuration

## `src/config.ts`

``` typescript
import dotenv from "dotenv";

dotenv.config();

function required(name: string): string {
  const value = process.env[name];

  if (!value) {
    throw new Error(`${name} is not configured`);
  }

  return value;
}

export const config = {
  port: Number(process.env.PORT ?? 3000),

  azure: {
    endpoint: required(
      "AZURE_DOCUMENT_INTELLIGENCE_ENDPOINT"
    ),

    key: required(
      "AZURE_DOCUMENT_INTELLIGENCE_KEY"
    )
  },

  openai: {
    apiKey: required(
      "OPENAI_API_KEY"
    ),

    model:
      process.env.OPENAI_MODEL ??
      "gpt-5.6-luna"
  },

  classification: {
    deterministicThreshold:
      Number(
        process.env.DETERMINISTIC_THRESHOLD ??
        0.90
      ),

    aiThreshold:
      Number(
        process.env.AI_THRESHOLD ??
        0.85
      )
  }
};
```

------------------------------------------------------------------------

# 7. Types

## `src/types.ts`

``` typescript
export type DocumentType =
  | "TAX_RETURN"
  | "TAX_COMPUTATION"
  | "UNKNOWN";

export type ClassificationMethod =
  | "DETERMINISTIC"
  | "AI"
  | "REVIEW";

export interface Page {
  pageNumber: number;
  text: string;
}

export interface ClassificationResult {
  documentType:
    | "TAX_RETURN"
    | "TAX_COMPUTATION"
    | "UNKNOWN";

  confidence: number;

  evidence: string[];

  pages: number[];

  method: ClassificationMethod;
}

export interface DocumentSegment {
  documentType:
    | "TAX_RETURN"
    | "TAX_COMPUTATION";

  pages: number[];

  confidence: number;

  evidence: string[];

  method: ClassificationMethod;
}

export interface DocumentClassification {
  documents: DocumentSegment[];

  reviewRequired: boolean;

  reviewReason?: string;
}

export interface YearAnalysis {
  fileName: string;

  pageCount: number;

  classification: DocumentClassification;
}

export interface ExecutionResult {
  executionId: string;

  status:
    | "READY"
    | "REVIEW_REQUIRED";

  currentYear: YearAnalysis;

  previousYear: YearAnalysis;

  validation: {
    currentYearTaxReturn: boolean;

    currentYearTaxComputation: boolean;

    previousYearTaxReturn: boolean;

    previousYearTaxComputation: boolean;
  };

  readyToRun: boolean;
}
```

------------------------------------------------------------------------

# 8. Azure Document Intelligence

## `src/azure/document-intelligence.service.ts`

The service sends the PDF to Azure Document Intelligence and polls the
asynchronous operation until processing finishes.

``` typescript
import DocumentIntelligence
  from "@azure-rest/ai-document-intelligence";

import {
  AzureKeyCredential
} from "@azure/core-auth";

import fs from "fs";

import { config } from "../config";

const client =
  DocumentIntelligence(
    config.azure.endpoint,
    new AzureKeyCredential(
      config.azure.key
    )
  );

export async function analyzePdf(
  filePath: string
) {
  const buffer =
    fs.readFileSync(
      filePath
    );

  const response =
    await client
      .path(
        "/documentModels/{modelId}:analyze",
        "prebuilt-layout"
      )
      .post({
        contentType:
          "application/pdf",

        body:
          buffer
      });

  if (
    response.status !== 202
  ) {
    throw new Error(
      `Azure Document Intelligence failed: ${response.status}`
    );
  }

  const operationLocation =
    response.headers[
      "operation-location"
    ];

  if (!operationLocation) {
    throw new Error(
      "Azure operation-location was not returned"
    );
  }

  while (true) {
    const result =
      await fetch(
        operationLocation
      );

    if (!result.ok) {
      throw new Error(
        `Azure polling failed: ${result.status}`
      );
    }

    const data =
      await result.json();

    if (
      data.status ===
      "succeeded"
    ) {
      return data;
    }

    if (
      data.status ===
      "failed"
    ) {
      throw new Error(
        "Azure Document Intelligence analysis failed"
      );
    }

    await new Promise(
      resolve =>
        setTimeout(
          resolve,
          1000
        )
    );
  }
}
```

------------------------------------------------------------------------

# 9. Extract Pages

## `src/azure/page-extractor.ts`

The classifier needs page-level information because a PDF may contain
multiple document types.

``` typescript
import {
  Page
} from "../types";

export function extractPages(
  analysis: any
): Page[] {
  const pages =
    analysis
      ?.analyzeResult
      ?.pages ?? [];

  return pages.map(
    (
      page: any,
      index: number
    ) => {
      const lines =
        page.lines ?? [];

      const text =
        lines
          .map(
            (line: any) =>
              line.content
          )
          .join(" ");

      return {
        pageNumber:
          index + 1,

        text
      };
    }
  );
}
```

------------------------------------------------------------------------

# 10. Classification Rules

## `src/classification/rules.ts`

The rule weight represents how strongly a phrase indicates a document
type.

For example:

-   `SA100` is a very strong Tax Return indicator.
-   `SA110` is a very strong Tax Computation indicator.
-   Generic terms such as `tax calculation` are strong but less
    specific.
-   Supporting terms such as `employment` or `capital gains` have lower
    weight.

``` typescript
export interface ClassificationRule {
  phrase: string;

  weight: number;

  documentType:
    | "TAX_RETURN"
    | "TAX_COMPUTATION";
}

export const rules:
  ClassificationRule[] = [

  // -----------------------------
  // TAX RETURN
  // -----------------------------

  {
    phrase: "sa100",
    weight: 100,
    documentType:
      "TAX_RETURN"
  },

  {
    phrase:
      "self assessment tax return",
    weight: 100,
    documentType:
      "TAX_RETURN"
  },

  {
    phrase:
      "tax return",
    weight: 60,
    documentType:
      "TAX_RETURN"
  },

  {
    phrase:
      "self employment",
    weight: 20,
    documentType:
      "TAX_RETURN"
  },

  {
    phrase:
      "employment",
    weight: 10,
    documentType:
      "TAX_RETURN"
  },

  {
    phrase:
      "uk property",
    weight: 20,
    documentType:
      "TAX_RETURN"
  },

  {
    phrase:
      "foreign income",
    weight: 20,
    documentType:
      "TAX_RETURN"
  },

  {
    phrase:
      "capital gains",
    weight: 20,
    documentType:
      "TAX_RETURN"
  },

  // -----------------------------
  // TAX COMPUTATION
  // -----------------------------

  {
    phrase:
      "sa110",
    weight: 100,
    documentType:
      "TAX_COMPUTATION"
  },

  {
    phrase:
      "tax computation",
    weight: 100,
    documentType:
      "TAX_COMPUTATION"
  },

  {
    phrase:
      "tax calculation summary",
    weight: 90,
    documentType:
      "TAX_COMPUTATION"
  },

  {
    phrase:
      "tax calculation",
    weight: 70,
    documentType:
      "TAX_COMPUTATION"
  },

  {
    phrase:
      "taxable income",
    weight: 30,
    documentType:
      "TAX_COMPUTATION"
  },

  {
    phrase:
      "income tax charged",
    weight: 40,
    documentType:
      "TAX_COMPUTATION"
  },

  {
    phrase:
      "payments on account",
    weight: 40,
    documentType:
      "TAX_COMPUTATION"
  },

  {
    phrase:
      "balancing payment",
    weight: 40,
    documentType:
      "TAX_COMPUTATION"
  },

  {
    phrase:
      "total tax due",
    weight: 40,
    documentType:
      "TAX_COMPUTATION"
  }
];
```

------------------------------------------------------------------------

# 11. Deterministic Classifier

## `src/classification/deterministic-classifier.ts`

The deterministic classifier:

1.  Normalizes extracted text.
2.  Checks configured rules.
3.  Calculates weighted scores.
4.  Collects evidence.
5.  Records page numbers.
6.  Calculates confidence.

``` typescript
import {
  Page,
  ClassificationResult
} from "../types";

import {
  rules
} from "./rules";

function normalize(
  text: string
): string {
  return text
    .toLowerCase()
    .replace(/\s+/g, " ")
    .trim();
}

export function classifyDeterministically(
  pages: Page[]
): ClassificationResult {

  let taxReturnScore = 0;

  let taxComputationScore = 0;

  const returnEvidence:
    string[] = [];

  const computationEvidence:
    string[] = [];

  const returnPages:
    number[] = [];

  const computationPages:
    number[] = [];

  for (
    const page of pages
  ) {
    const text =
      normalize(
        page.text
      );

    let pageReturnScore = 0;

    let pageComputationScore = 0;

    for (
      const rule of rules
    ) {
      if (
        text.includes(
          rule.phrase
        )
      ) {
        if (
          rule.documentType ===
          "TAX_RETURN"
        ) {
          taxReturnScore +=
            rule.weight;

          pageReturnScore +=
            rule.weight;

          returnEvidence.push(
            `Page ${page.pageNumber}: ${rule.phrase}`
          );
        }

        if (
          rule.documentType ===
          "TAX_COMPUTATION"
        ) {
          taxComputationScore +=
            rule.weight;

          pageComputationScore +=
            rule.weight;

          computationEvidence.push(
            `Page ${page.pageNumber}: ${rule.phrase}`
          );
        }
      }
    }

    if (
      pageReturnScore > 0
    ) {
      returnPages.push(
        page.pageNumber
      );
    }

    if (
      pageComputationScore > 0
    ) {
      computationPages.push(
        page.pageNumber
      );
    }
  }

  const total =
    taxReturnScore +
    taxComputationScore;

  if (total === 0) {
    return {
      documentType:
        "UNKNOWN",

      confidence:
        0,

      evidence: [],

      pages: [],

      method:
        "REVIEW"
    };
  }

  const winningScore =
    Math.max(
      taxReturnScore,
      taxComputationScore
    );

  const confidence =
    winningScore /
    total;

  if (
    taxReturnScore >
    taxComputationScore
  ) {
    return {
      documentType:
        "TAX_RETURN",

      confidence,

      evidence:
        [...new Set(
          returnEvidence
        )],

      pages:
        [...new Set(
          returnPages
        )],

      method:
        "DETERMINISTIC"
    };
  }

  if (
    taxComputationScore >
    taxReturnScore
  ) {
    return {
      documentType:
        "TAX_COMPUTATION",

      confidence,

      evidence:
        [...new Set(
          computationEvidence
        )],

      pages:
        [...new Set(
          computationPages
        )],

      method:
        "DETERMINISTIC"
    };
  }

  return {
    documentType:
      "UNKNOWN",

    confidence:
      0.5,

    evidence: [
      ...returnEvidence,
      ...computationEvidence
    ],

    pages: [
      ...new Set([
        ...returnPages,
        ...computationPages
      ])
    ],

    method:
      "REVIEW"
  };
}
```

------------------------------------------------------------------------

# 12. AI Fallback

## `src/classification/ai-classifier.ts`

The AI classifier is not the first step.

It is used only when deterministic rules cannot classify the document
with sufficient confidence.

``` typescript
import OpenAI from "openai";

import {
  z
} from "zod";

import {
  Page
} from "../types";

import {
  config
} from "../config";

const openai =
  new OpenAI({
    apiKey:
      config.openai.apiKey
  });

const AIResultSchema =
  z.object({

    documentType:
      z.enum([
        "TAX_RETURN",
        "TAX_COMPUTATION",
        "UNKNOWN"
      ]),

    confidence:
      z.number()
        .min(0)
        .max(1),

    evidence:
      z.array(
        z.string()
      ),

    pages:
      z.array(
        z.number()
      )
  });

export async function classifyWithAI(
  pages: Page[]
) {

  const content =
    pages
      .map(
        page =>
          `
PAGE ${page.pageNumber}

${page.text}
`
      )
      .join("\n");

  const response =
    await openai.responses.parse({

      model:
        config.openai.model,

      input: [

        {
          role:
            "system",

          content: `
You are a UK tax document classification system.

Classify the supplied document pages into:

TAX_RETURN
TAX_COMPUTATION
UNKNOWN

TAX_RETURN means the pages represent a
Self Assessment tax return or its return
sections/schedules.

TAX_COMPUTATION means the pages represent
a calculation of the tax liability, including
tax calculation, tax liability, payments on
account, balancing payment, etc.

UNKNOWN means there is insufficient evidence.

Rules:

1. Use only the supplied document text.
2. Do not invent evidence.
3. Prefer explicit document titles,
   form numbers and headings.
4. Consider multiple pages together.
5. Identify the pages supporting your decision.
6. Confidence must be between 0 and 1.
7. If evidence is weak or contradictory,
   return UNKNOWN.
          `
        },

        {
          role:
            "user",

          content
        }
      ],

      text: {

        format: {

          type:
            "json_schema",

          name:
            "tax_document_classification",

          strict:
            true,

          schema: {

            type:
              "object",

            properties: {

              documentType: {

                type:
                  "string",

                enum: [
                  "TAX_RETURN",
                  "TAX_COMPUTATION",
                  "UNKNOWN"
                ]
              },

              confidence: {

                type:
                  "number"
              },

              evidence: {

                type:
                  "array",

                items: {
                  type:
                    "string"
                }
              },

              pages: {

                type:
                  "array",

                items: {
                  type:
                    "number"
                }
              }
            },

            required: [
              "documentType",
              "confidence",
              "evidence",
              "pages"
            ],

            additionalProperties:
              false
          }
        }
      }
    });

  const parsed =
    response.output_parsed;

  if (!parsed) {
    throw new Error(
      "AI returned no parsed result"
    );
  }

  return AIResultSchema.parse(
    parsed
  );
}
```

------------------------------------------------------------------------

# 13. Hybrid Classifier

## `src/classification/hybrid-classifier.ts`

This is where the AI fallback is actually invoked.

The decision flow is:

``` text
Deterministic
      |
      +-- confidence >= 0.90 --> FINAL
      |
      +-- confidence < 0.90 --> AI
                                  |
                                  +-- confidence >= 0.85 --> FINAL
                                  |
                                  +-- confidence < 0.85 --> REVIEW
```

Implementation:

``` typescript
import {
  Page,
  DocumentClassification
} from "../types";

import {
  config
} from "../config";

import {
  classifyDeterministically
} from "./deterministic-classifier";

import {
  classifyWithAI
} from "./ai-classifier";

export async function classifyDocument(
  pages: Page[]
): Promise<DocumentClassification> {

  // --------------------------------
  // 1. Deterministic classification
  // --------------------------------

  const deterministic =
    classifyDeterministically(
      pages
    );

  console.log(
    "Deterministic result:",
    deterministic
  );

  // --------------------------------
  // 2. Strong deterministic result
  // --------------------------------

  if (
    deterministic.documentType !==
      "UNKNOWN" &&

    deterministic.confidence >=
      config.classification
        .deterministicThreshold
  ) {

    return {
      documents: [
        {
          documentType:
            deterministic.documentType,

          pages:
            deterministic.pages,

          confidence:
            deterministic.confidence,

          evidence:
            deterministic.evidence,

          method:
            "DETERMINISTIC"
        }
      ],

      reviewRequired:
        false
    };
  }

  // --------------------------------
  // 3. AI fallback
  // --------------------------------

  console.log(
    "Deterministic result ambiguous."
  );

  console.log(
    "Calling AI fallback..."
  );

  try {

    const ai =
      await classifyWithAI(
        pages
      );

    console.log(
      "AI result:",
      ai
    );

    // --------------------------------
    // 4. AI strong enough
    // --------------------------------

    if (
      ai.documentType !==
        "UNKNOWN" &&

      ai.confidence >=
        config.classification
          .aiThreshold
    ) {

      return {
        documents: [
          {
            documentType:
              ai.documentType,

            pages:
              ai.pages,

            confidence:
              ai.confidence,

            evidence:
              ai.evidence,

            method:
              "AI"
          }
        ],

        reviewRequired:
          false
      };
    }

    // --------------------------------
    // 5. AI still uncertain
    // --------------------------------

    return {
      documents: [],

      reviewRequired:
        true,

      reviewReason:
        "Both deterministic and AI classification were below the required confidence."
    };

  } catch (error) {

    console.error(
      "AI fallback failed:",
      error
    );

    return {
      documents: [],

      reviewRequired:
        true,

      reviewReason:
        "AI classification failed."
    };
  }
}
```

------------------------------------------------------------------------

# 14. Analyze a Single PDF

## `src/services/document.service.ts`

``` typescript
import {
  analyzePdf
} from "../azure/document-intelligence.service";

import {
  extractPages
} from "../azure/page-extractor";

import {
  classifyDocument
} from "../classification/hybrid-classifier";

import {
  YearAnalysis,
  ExecutionResult
} from "../types";

import path from "path";

function hasDocument(
  analysis: YearAnalysis,
  type:
    | "TAX_RETURN"
    | "TAX_COMPUTATION"
) {
  return analysis.classification.documents
    .some(
      document =>
        document.documentType === type
    );
}

export async function analyzeFile(
  filePath: string
): Promise<YearAnalysis> {

  console.log(
    `Analyzing ${filePath}`
  );

  // -----------------------------
  // Azure Document Intelligence
  // -----------------------------

  const analysis =
    await analyzePdf(
      filePath
    );

  // -----------------------------
  // Extract pages
  // -----------------------------

  const pages =
    extractPages(
      analysis
    );

  console.log(
    `Extracted ${pages.length} pages`
  );

  // -----------------------------
  // Hybrid classification
  // -----------------------------

  const classification =
    await classifyDocument(
      pages
    );

  return {

    fileName:
      path.basename(
        filePath
      ),

    pageCount:
      pages.length,

    classification
  };
}

export async function analyzeExecution(
  executionId: string,
  currentYearFile: string,
  previousYearFile: string
): Promise<ExecutionResult> {

  const currentYear =
    await analyzeFile(
      currentYearFile
    );

  const previousYear =
    await analyzeFile(
      previousYearFile
    );

  const validation = {

    currentYearTaxReturn:
      hasDocument(
        currentYear,
        "TAX_RETURN"
      ),

    currentYearTaxComputation:
      hasDocument(
        currentYear,
        "TAX_COMPUTATION"
      ),

    previousYearTaxReturn:
      hasDocument(
        previousYear,
        "TAX_RETURN"
      ),

    previousYearTaxComputation:
      hasDocument(
        previousYear,
        "TAX_COMPUTATION"
      )
  };

  const readyToRun =
    Object.values(
      validation
    ).every(Boolean) &&

    !currentYear.classification
      .reviewRequired &&

    !previousYear.classification
      .reviewRequired;

  return {

    executionId,

    status:
      readyToRun
        ? "READY"
        : "REVIEW_REQUIRED",

    currentYear,

    previousYear,

    validation,

    readyToRun
  };
}
```

------------------------------------------------------------------------

# 15. API Route

## `src/routes/document.routes.ts`

The endpoint accepts two files:

-   `currentYear`
-   `previousYear`

``` typescript
import {
  Router,
  Request,
  Response
} from "express";

import multer from "multer";

import fs from "fs";

import {
  analyzeExecution
} from "../services/document.service";

const router =
  Router();

const upload =
  multer({
    dest:
      "uploads/"
  });

router.post(
  "/analyze",

  upload.fields([
    {
      name:
        "currentYear",

      maxCount:
        1
    },

    {
      name:
        "previousYear",

      maxCount:
        1
    }
  ]),

  async (
    req: Request,
    res: Response
  ) => {

    try {

      const files =
        req.files as
        {
          [
            fieldName: string
          ]:
          Express.Multer.File[]
        };

      const current =
        files?.currentYear?.[0];

      const previous =
        files?.previousYear?.[0];

      if (
        !current ||
        !previous
      ) {

        return res
          .status(400)
          .json({

            error:
              "Both currentYear and previousYear PDF files are required."
          });
      }

      if (
        !current.originalname
          .toLowerCase()
          .endsWith(".pdf") ||

        !previous.originalname
          .toLowerCase()
          .endsWith(".pdf")
      ) {

        return res
          .status(400)
          .json({

            error:
              "Only PDF files are supported."
          });
      }

      const executionId =
        `EXEC-${Date.now()}`;

      const result =
        await analyzeExecution(
          executionId,
          current.path,
          previous.path
        );

      return res.json(
        result
      );

    } catch (error) {

      console.error(
        error
      );

      return res
        .status(500)
        .json({

          error:
            error instanceof Error
              ? error.message
              : "Unknown error"
        });

    } finally {

      const files =
        req.files as
        {
          [
            fieldName: string
          ]:
          Express.Multer.File[]
        };

      const uploadedFiles =
        Object.values(
          files ?? {}
        ).flat();

      for (
        const file of uploadedFiles
      ) {

        try {

          if (
            fs.existsSync(
              file.path
            )
          ) {
            fs.unlinkSync(
              file.path
            );
          }

        } catch {
          // Ignore cleanup errors
        }
      }
    }
  }
);

export default router;
```

------------------------------------------------------------------------

# 16. Server

## `src/server.ts`

``` typescript
import express from "express";

import {
  config
} from "./config";

import documentRoutes
  from "./routes/document.routes";

const app =
  express();

app.use(
  express.json()
);

app.get(
  "/health",
  (
    _req,
    res
  ) => {

    res.json({
      status:
        "ok"
    });
  }
);

app.use(
  "/api/documents",
  documentRoutes
);

app.listen(
  config.port,
  () => {

    console.log(
      `Server running on port ${config.port}`
    );
  }
);
```

------------------------------------------------------------------------

# 17. Create Upload Folder

From the project root:

``` bash
mkdir uploads
```

------------------------------------------------------------------------

# 18. Run the Application

Development:

``` bash
npm run dev
```

Expected:

``` text
Server running on port 3000
```

Production build:

``` bash
npm run build
```

Production start:

``` bash
npm start
```

------------------------------------------------------------------------

# 19. Health Check

Request:

``` text
GET http://localhost:3000/health
```

Expected response:

``` json
{
  "status": "ok"
}
```

------------------------------------------------------------------------

# 20. Analyze Current Year and Previous Year PDFs

Endpoint:

``` text
POST http://localhost:3000/api/documents/analyze
```

Use `multipart/form-data`.

Fields:

``` text
currentYear   File   CY.pdf
previousYear  File   PY.pdf
```

Example with curl:

``` bash
curl -X POST \
  http://localhost:3000/api/documents/analyze \
  -F "currentYear=@CY.pdf" \
  -F "previousYear=@PY.pdf"
```

------------------------------------------------------------------------

# 21. Example Successful Response

``` json
{
  "executionId": "EXEC-1759220000000",
  "status": "READY",

  "currentYear": {
    "fileName": "CY.pdf",
    "pageCount": 100,

    "classification": {
      "documents": [
        {
          "documentType": "TAX_RETURN",
          "pages": [1, 2, 3, 4, 5],
          "confidence": 0.97,
          "evidence": [
            "Page 1: sa100",
            "Page 1: self assessment tax return"
          ],
          "method": "DETERMINISTIC"
        },
        {
          "documentType": "TAX_COMPUTATION",
          "pages": [70, 71, 72, 73],
          "confidence": 0.94,
          "evidence": [
            "Page 70: tax calculation",
            "Page 71: payments on account"
          ],
          "method": "AI"
        }
      ],

      "reviewRequired": false
    }
  },

  "previousYear": {
    "fileName": "PY.pdf",
    "pageCount": 85,

    "classification": {
      "documents": [
        {
          "documentType": "TAX_RETURN",
          "pages": [1, 2, 3, 4],
          "confidence": 0.98,
          "evidence": [
            "Page 1: sa100"
          ],
          "method": "DETERMINISTIC"
        },
        {
          "documentType": "TAX_COMPUTATION",
          "pages": [60, 61, 62],
          "confidence": 0.96,
          "evidence": [
            "Page 60: sa110"
          ],
          "method": "DETERMINISTIC"
        }
      ],

      "reviewRequired": false
    }
  },

  "validation": {
    "currentYearTaxReturn": true,
    "currentYearTaxComputation": true,
    "previousYearTaxReturn": true,
    "previousYearTaxComputation": true
  },

  "readyToRun": true
}
```

------------------------------------------------------------------------

# 22. Example Review Required Response

If neither deterministic rules nor AI can confidently classify a
document:

``` json
{
  "executionId": "EXEC-1759220000000",
  "status": "REVIEW_REQUIRED",

  "validation": {
    "currentYearTaxReturn": false,
    "currentYearTaxComputation": false,
    "previousYearTaxReturn": true,
    "previousYearTaxComputation": true
  },

  "readyToRun": false
}
```

The process should not continue automatically until the
missing/ambiguous document is resolved.

------------------------------------------------------------------------

# 23. How the Rule Weight Works

The rule weight represents the strength of evidence.

For example:

``` text
SA100                         100
Self Assessment Tax Return    100
Tax Return                     60
Capital Gains                  20
Employment                     10
```

For Tax Computation:

``` text
SA110                         100
Tax Computation               100
Tax Calculation Summary        90
Tax Calculation                70
Payments on Account            40
Taxable Income                 30
```

Suppose a page contains:

``` text
SA100
Self Assessment Tax Return
Employment
Capital Gains
```

The Tax Return score would be:

``` text
100 + 100 + 10 + 20
= 230
```

If another page contains:

``` text
Tax Calculation
Payments on Account
Total Tax Due
```

the Tax Computation score could be:

``` text
70 + 40 + 40
= 150
```

The current implementation uses the total scores to calculate a simple
confidence:

``` text
confidence =
winningScore / (taxReturnScore + taxComputationScore)
```

This is useful as a baseline, but it should eventually be replaced with
a more robust calibrated scoring model.

------------------------------------------------------------------------

# 24. Why Deterministic Classification Comes First

The system intentionally does not send every document directly to an
LLM.

The preferred flow is:

``` text
                 PDF
                  |
                  v
        Azure Document Intelligence
                  |
                  v
          Extracted page text
                  |
                  v
        Deterministic classifier
             /           \
            /             \
       confident       ambiguous
          |                 |
          v                 v
        FINAL            AI fallback
                            |
                     /--------------\
                    /                \
               confident          uncertain
                   |                  |
                   v                  v
                 FINAL             REVIEW
```

Advantages:

-   predictable
-   explainable
-   cheaper
-   faster
-   easier to test
-   easier to audit
-   reduces unnecessary LLM calls

------------------------------------------------------------------------

# 25. Where the AI Fallback Happens

The AI fallback is called here:

``` typescript
if (
  deterministic.documentType !==
    "UNKNOWN" &&

  deterministic.confidence >=
    config.classification
      .deterministicThreshold
) {
  return finalResult;
}

const ai =
  await classifyWithAI(
    pages
  );
```

Therefore:

``` text
Deterministic confidence >= 0.90
        |
        +--> Do NOT call AI

Deterministic confidence < 0.90
        |
        +--> Call AI
```

The AI result is then checked against:

``` text
AI_THRESHOLD=0.85
```

So:

``` text
AI confidence >= 0.85
        |
        +--> Accept

AI confidence < 0.85
        |
        +--> REVIEW_REQUIRED
```

------------------------------------------------------------------------

# 26. Important Limitation of the Baseline Implementation

The baseline implementation sends all extracted pages to the classifier.

For example:

``` text
100-page PDF
     |
     v
Azure Document Intelligence
     |
     v
100 page texts
     |
     v
AI fallback
```

This works as a simple MVP but is not the ideal production design.

A better production architecture is:

``` text
100-page PDF
     |
     v
Azure Document Intelligence
     |
     v
100 page results
     |
     v
Page segmentation
     |
     +---- likely Tax Return pages
     |
     +---- likely Tax Computation pages
     |
     +---- ambiguous pages
                  |
                  v
             AI fallback
```

Only ambiguous/candidate sections should be sent to the AI model.

------------------------------------------------------------------------

# 27. Recommended Production Architecture

``` text
                    CY PDF
                      |
                      v
            Azure Document Intelligence
                      |
                      v
                 Page Results
                      |
                      v
              Page Segmentation
                      |
        +-------------+-------------+
        |             |             |
        v             v             v
   Tax Return    Tax Computation   Unknown
    candidates     candidates       pages
        |             |             |
        +-------------+-------------+
                      |
                      v
             Deterministic Rules
                      |
             +--------+--------+
             |                 |
          confident         ambiguous
             |                 |
             v                 v
           FINAL          AI fallback
                               |
                        +------+------+
                        |             |
                     confident      low
                        |             |
                        v             v
                      FINAL        REVIEW
```

------------------------------------------------------------------------

# 28. Page Segmentation

A production classifier should detect boundaries between document
sections.

For example:

``` text
Pages 1-45
    -> Tax Return

Pages 46-55
    -> Supplementary schedules

Pages 56-60
    -> Tax Computation

Pages 61-100
    -> Supporting documents
```

Instead of treating the entire PDF as one document, the system should
produce:

``` json
{
  "documents": [
    {
      "documentType": "TAX_RETURN",
      "pages": [1, 2, 3, 4, 5]
    },
    {
      "documentType": "TAX_COMPUTATION",
      "pages": [56, 57, 58, 59, 60]
    }
  ]
}
```

This is important because a single uploaded PDF may contain multiple
logical documents.

------------------------------------------------------------------------

# 29. Tax Return vs Tax Computation

Conceptually:

## Tax Return

The Tax Return represents information submitted for the taxpayer.

Typical information includes:

``` text
Personal information
Employment income
Self-employment
Property income
Foreign income
Capital gains
Pension contributions
Interest
Dividends
Other income
Allowances
Reliefs
```

## Tax Computation

The Tax Computation explains how the tax liability was calculated.

Typical information includes:

``` text
Total income
Taxable income
Income tax
Dividend tax
Capital gains tax
National Insurance
Tax deducted
Payments on account
Balancing payment
Total tax due
```

Therefore, the two documents should not be treated as interchangeable.

------------------------------------------------------------------------

# 30. Why Form Numbers Are Strong Evidence

Form numbers are generally more reliable than generic words.

For example:

``` text
SA100
```

is a much stronger indicator of a Tax Return than:

``` text
income
```

Similarly:

``` text
SA110
```

is a stronger indicator of a Tax Computation than:

``` text
tax
```

Therefore, production rules should prioritize:

1.  Form number
2.  Explicit document title
3.  Section heading
4.  Strong domain-specific phrase
5.  Generic supporting phrase

------------------------------------------------------------------------

# 31. Suggested Classification Evidence

Every classification should store evidence.

Example:

``` json
{
  "documentType": "TAX_RETURN",
  "confidence": 0.97,
  "method": "DETERMINISTIC",
  "pages": [1, 2, 3],
  "evidence": [
    "Page 1: SA100",
    "Page 1: Self Assessment Tax Return",
    "Page 2: Employment"
  ]
}
```

This is important for auditability.

A user reviewing a classification can understand:

``` text
Why did the system classify this as Tax Return?
```

rather than seeing only:

``` text
TAX_RETURN
```

------------------------------------------------------------------------

# 32. Current Year / Previous Year Validation

Before running the comparison process, validate:

``` text
CY
 |
 +-- Tax Return
 |
 +-- Tax Computation

PY
 |
 +-- Tax Return
 |
 +-- Tax Computation
```

The process is ready only when all required components exist:

``` text
CY Tax Return             YES
CY Tax Computation        YES

PY Tax Return             YES
PY Tax Computation        YES

Review Required           NO
```

Then:

``` text
READY_TO_RUN = true
```

Otherwise:

``` text
READY_TO_RUN = false
```

------------------------------------------------------------------------

# 33. Example Business Workflow

A tax professional uploads:

``` text
CY.pdf
PY.pdf
```

The system performs:

``` text
1. Upload CY
2. Upload PY

3. Azure Document Intelligence
4. Extract page-level text

5. Identify document sections

6. Run deterministic classification

7. If ambiguous:
      run AI fallback

8. Validate CY:
      Tax Return?
      Tax Computation?

9. Validate PY:
      Tax Return?
      Tax Computation?

10. If everything is valid:
      READY_TO_RUN

11. Start datapoint extraction

12. Compare CY vs PY
```

------------------------------------------------------------------------

# 34. Future Datapoint Comparison

Once document classification is complete, the next stage can use the
identified document sections.

For example:

``` text
CY Tax Return
      |
      +-- Employment Income
      +-- Property Income
      +-- Dividend Income
      +-- Pension Contribution
      +-- Capital Gains

PY Tax Return
      |
      +-- Employment Income
      +-- Property Income
      +-- Dividend Income
      +-- Pension Contribution
      +-- Capital Gains
```

Then comparison rules can map:

``` text
PY Employment Income
          |
          v
CY Employment Income
```

and calculate:

``` text
CY value
PY value
Difference
Percentage change
```

------------------------------------------------------------------------

# 35. Example Datapoint

A datapoint could be represented as:

``` json
{
  "name": "Employment Income",
  "dataType": "INTEGER",
  "documentType": "TAX_RETURN",
  "section": "Employment"
}
```

Another could be:

``` json
{
  "name": "Employment Details",
  "dataType": "LIST",
  "documentType": "TAX_RETURN",
  "section": "Employment"
}
```

For list-based datapoints, the extraction prompt should define the
expected list structure.

------------------------------------------------------------------------

# 36. Important Production Improvements

The MVP should eventually be enhanced with:

## 36.1 Page segmentation

Do not classify the entire PDF as a single document.

## 36.2 OCR handling

Some PDFs may be scanned documents.

The extraction pipeline should support OCR.

## 36.3 Better rule management

Rules should eventually be stored in a database rather than hard-coded.

Possible schema:

``` text
classification_rules

id
document_type
phrase
weight
rule_type
active
created_at
updated_at
```

## 36.4 Rule versioning

Classification rules can change over time.

Store:

``` text
rule_version
```

with every execution.

## 36.5 AI prompt versioning

Store the AI prompt version with each classification.

Example:

``` text
prompt_version = tax-document-classifier-v3
```

## 36.6 Audit trail

Store:

``` text
execution_id
file_id
page_number
classification
confidence
method
evidence
rule_version
prompt_version
created_at
```

## 36.7 Human review

If classification is uncertain:

``` text
REVIEW_REQUIRED
```

The UI can show:

``` text
Document classification requires review.

Possible document:
Tax Computation

Confidence:
0.68

Evidence:
Page 72: tax calculation
Page 73: total tax due
```

The user can then manually select:

``` text
Tax Return
Tax Computation
Unknown
```

------------------------------------------------------------------------

# 37. Recommended Final Architecture

``` text
                    React UI
                       |
                       v
                Node.js API
                       |
                       v
                Upload Service
                       |
                       v
                 File Storage
                       |
                       v
           Azure Document Intelligence
                       |
                       v
                Page Extraction
                       |
                       v
                Page Segmentation
                       |
                       v
             Deterministic Rules
                       |
              +--------+--------+
              |                 |
          confident          ambiguous
              |                 |
              v                 v
            FINAL          AI Classifier
                                |
                         +------+------+
                         |             |
                     confident       uncertain
                         |             |
                         v             v
                       FINAL        HUMAN REVIEW
                         |
                         v
                 Document Validation
                         |
                         v
                    CY / PY Ready
                         |
                         v
                 Datapoint Extraction
                         |
                         v
                   Comparison Engine
                         |
                         v
                  Results / Reports
```

------------------------------------------------------------------------

# 38. Summary

The core implementation uses a hybrid strategy:

``` text
Azure Document Intelligence
          |
          v
Page-level extraction
          |
          v
Deterministic classification
          |
          +---- high confidence ---> final
          |
          +---- ambiguous ---------> AI
                                       |
                                       +---- high confidence --> final
                                       |
                                       +---- low confidence ---> review
```

The important design principle is:

> Use deterministic, explainable rules whenever possible and use AI as a
> fallback for ambiguity rather than making every classification an LLM
> decision.

This gives the system a practical balance between:

-   accuracy
-   cost
-   speed
-   explainability
-   auditability
-   human review

------------------------------------------------------------------------

# 39. Next Development Step

The next implementation step should be **page-level document
segmentation**.

Instead of:

``` text
PDF
 |
 v
Classify entire PDF
```

move to:

``` text
PDF
 |
 v
Azure Document Intelligence
 |
 v
Pages
 |
 v
Detect document boundaries
 |
 +---- Tax Return
 |
 +---- Tax Computation
 |
 +---- Other/supporting documents
 |
 v
Run deterministic classification per segment
 |
 v
AI only for ambiguous segments
```

This is the architecture that should be used before building the full
production-grade tax comparison pipeline.
