# Configuration-as-Data Architecture - EVA-JP-v1.2

**Date**: February 4, 2026  
**Project**: EVA-JP-v1.2 APIM Integration + Multi-Project Support  
**Phase**: Post-Analysis Design - Configuration Management  
**Status**: Design Complete - Ready for Implementation

---

## Executive Summary

**Vision**: Eliminate all hardcoded literals (cost headers, system prompts, feature flags, limits) and enable project administrators to manage configurations through UI, with support for multi-project deployments.

**Impact**:
- **Flexibility**: Project admins deploy new projects without code changes
- **Cost Attribution**: Automatic FinOps metadata in every API call
- **Multi-Tenancy**: Support multiple projects with isolated configurations
- **Observability**: Track costs by client/project/phase/task/environment

**Timeline**: 4 weeks (Phase 4-8 APIM implementation + Config Management)

---

## Current State Analysis

### Problem 1: Hardcoded Cost Attribution Headers

**Location**: `app/frontend/src/api/api.ts`

```typescript
// ❌ CURRENT - Hardcoded literals
const headers = {
  'X-Client': 'ESDC-IT',              // Hardcoded
  'X-Project': 'eva-jp-v1.2',         // Hardcoded
  'X-Environment': 'dev',             // Hardcoded
  'X-CostCenter': 'AICOE-123',        // Hardcoded
  'X-Phase': 'Phase2',                // Hardcoded
  'X-Task': 'chat-query',             // Hardcoded
};
```

**Issue**: Cannot support multiple projects, cannot change attribution without code deployment.

---

### Problem 2: Hardcoded System Prompts

**Location**: `app/backend/approaches/chatreadretrieveread.py`

```python
# ❌ CURRENT - Hardcoded in code
SYSTEM_MESSAGE = """You are an AI assistant helping with Employment Insurance questions.
Answer based on the following sources:
{sources}
"""
```

**Issue**: Changing prompts requires backend deployment. Cannot A/B test prompts. Cannot customize per project.

---

### Problem 3: Hardcoded Feature Flags & Limits

**Location**: `app/backend/backend.env`

```bash
# ❌ CURRENT - Hardcoded in environment file
ENABLE_MATH_ASSISTANT=true
ENABLE_UNGROUNDED_CHAT=true
MAX_CSV_FILE_SIZE=20
MAX_URLS_TO_SCRAPE=200
```

**Issue**: Requires environment file changes + restart. Cannot toggle features per project.

---

## Target Architecture

### Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     Frontend (React)                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  ConfigService.loadConfig()                            │ │
│  │    → GET /api/project-config                           │ │
│  │    → Returns: cost_attribution, feature_flags, limits  │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────┬───────────────────────────┘
                                  │ HTTP + Cost Headers
                                  ↓
┌─────────────────────────────────────────────────────────────┐
│                      APIM Gateway                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Inbound Policy:                                       │ │
│  │    1. Validate cost headers (X-Client, X-Project, etc)│ │
│  │    2. Log to Application Insights with FinOps dims     │ │
│  │    3. Forward to backend                               │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────┬───────────────────────────┘
                                  │
                                  ↓
┌─────────────────────────────────────────────────────────────┐
│                    Backend (FastAPI)                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  ConfigurationService                                  │ │
│  │    → Loads from Cosmos DB project_configuration       │ │
│  │    → 5-minute cache                                    │ │
│  │    → Returns: prompts, flags, limits, RBAC             │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────┬───────────────────────────┘
                                  │
                                  ↓
┌─────────────────────────────────────────────────────────────┐
│                 Cosmos DB (UserInformation)                  │
│  Containers:                                                 │
│    • project_configuration (NEW)                             │
│    • user_profiles (EXISTING - add project mapping)         │
│    • group_management (EXISTING)                             │
│    • chat_history_session (EXISTING)                         │
└─────────────────────────────────────────────────────────────┘
```

---

## Data Model Design

### Container 1: `project_configuration`

**Purpose**: Store all project-specific configuration (cost attribution, prompts, features, limits)

**Partition Key**: `/id` (project identifier)

**Schema**:

```json
{
  "id": "eva-jp-v1.2",
  "type": "project_config",
  "partition_key": "eva-jp-v1.2",
  
  "project_metadata": {
    "display_name": "EVA Jurisprudence Assistant v1.2",
    "description": "Employment Insurance case law search system",
    "owner": "marco.presta@hrsdc-rhdcc.gc.ca",
    "created_date": "2024-01-15T00:00:00Z",
    "last_modified": "2026-02-04T14:30:00Z",
    "status": "active",
    "version": "1.2.0"
  },
  
  "cost_attribution": {
    "client": "ESDC-IT",
    "project_code": "eva-jp-v1.2",
    "cost_center": "AICOE-123",
    "business_unit": "ESDC-IT",
    "environment": "dev",
    "phase": "Phase2-APIM-Integration",
    "default_task": "chat-query"
  },
  
  "system_prompts": {
    "default": "You are an AI assistant helping with Employment Insurance questions. Answer based on the following sources:\n{sources}\n\nCITICAL: Always cite sources using [doc0], [doc1] format.",
    "ungrounded": "You are a helpful AI assistant. Provide general guidance on Employment Insurance topics. Note: This response is not based on specific documents.",
    "math_assistant": "You are a mathematical assistant. Help users with calculations related to EI benefits, contribution rates, and weekly benefit amounts.",
    "tabular_data": "You are a data analysis assistant. Help users understand tables and charts related to EI data, statistics, and trends.",
    "translation": "Translate the following content while preserving technical EI terminology and legal accuracy."
  },
  
  "search_configuration": {
    "index_name": "index-jurisprudence",
    "semantic_ranker_enabled": true,
    "top_k_results": 5,
    "hybrid_search_alpha": 0.5,
    "temperature": 0.3,
    "excluded_folders": [],
    "excluded_tags": [],
    "allowed_content_types": ["pdf", "docx", "txt", "html"]
  },
  
  "feature_flags": {
    "enable_web_chat": false,
    "enable_ungrounded_chat": true,
    "enable_math_assistant": true,
    "enable_tabular_data_assistant": true,
    "enable_multimedia": false,
    "enable_bing_safe_search": true,
    "enable_dev_code": false,
    "enable_translation": true,
    "enable_url_scraping": true
  },
  
  "limits": {
    "max_csv_file_size_mb": 20,
    "max_urls_to_scrape": 200,
    "max_url_depth": 3,
    "max_tokens_per_request": 4000,
    "rate_limit_requests_per_minute": 30,
    "max_concurrent_sessions": 10,
    "session_timeout_minutes": 30
  },
  
  "rbac": {
    "admin_groups": ["9f540c2e-c6df-4c38-8a4f-7f2e0f8e5b4d"],
    "contributor_groups": ["3fece663-1f5e-4c38-9a4f-8f3e0f9e6c5e"],
    "reader_groups": ["7a8b9c0d-2e6f-5d49-ab5g-9g4f0g0f7d6f"],
    "enable_rbac": true,
    "default_role": "reader"
  },
  
  "azure_services": {
    "openai": {
      "deployment_name": "gpt-4o",
      "model_name": "gpt-4o",
      "model_version": "2024-11-20",
      "embedding_deployment": "text-embedding-ada-002"
    },
    "search": {
      "service_name": "marco-sandbox-search",
      "index_name": "index-jurisprudence"
    },
    "cosmos": {
      "database_name": "UserInformation",
      "containers": {
        "chat_history": "chat_history_session",
        "group_management": "group_management"
      }
    },
    "storage": {
      "account_name": "marcosand20260203",
      "container_name": "documents"
    }
  },
  
  "ui_customization": {
    "application_title": "EVA Jurisprudence Assistant",
    "warning_banner_text": "Marco Sandbox Environment - Dev2 Data",
    "hint_text": "Ask me about Employment Insurance case law, decisions, and regulations.",
    "theme_color": "#0078D4"
  }
}
```

**Indexing Policy**:

```json
{
  "indexingMode": "consistent",
  "automatic": true,
  "includedPaths": [
    { "path": "/id/*" },
    { "path": "/type/*" },
    { "path": "/project_metadata/status/*" },
    { "path": "/project_metadata/owner/*" },
    { "path": "/cost_attribution/*" }
  ],
  "excludedPaths": [
    { "path": "/system_prompts/*" },
    { "path": "/*" }
  ]
}
```

---

### Container 2: `user_profiles` (Enhanced)

**Purpose**: Store user profile with project access and preferences

**Partition Key**: `/user_id`

**Schema**:

```json
{
  "id": "marco.presta@hrsdc-rhdcc.gc.ca",
  "type": "user_profile",
  "user_id": "marco.presta@hrsdc-rhdcc.gc.ca",
  "partition_key": "marco.presta@hrsdc-rhdcc.gc.ca",
  
  "user_metadata": {
    "display_name": "Marco Presta",
    "email": "marco.presta@hrsdc-rhdcc.gc.ca",
    "department": "ESDC-IT",
    "title": "Senior Developer",
    "preferred_language": "en"
  },
  
  "current_project": "eva-jp-v1.2",
  
  "available_projects": [
    {
      "project_id": "eva-jp-v1.2",
      "display_name": "EVA Jurisprudence Assistant",
      "role": "admin",
      "granted_date": "2024-01-15T00:00:00Z",
      "last_accessed": "2026-02-04T14:30:00Z",
      "access_count": 1247,
      "is_favorite": true
    },
    {
      "project_id": "assist-me",
      "display_name": "AssistMe Knowledge Base",
      "role": "contributor",
      "granted_date": "2025-06-01T00:00:00Z",
      "last_accessed": "2026-02-01T10:15:00Z",
      "access_count": 85,
      "is_favorite": false
    },
    {
      "project_id": "ei-dsst",
      "display_name": "EI DSST Chatbot",
      "role": "reader",
      "granted_date": "2025-12-15T00:00:00Z",
      "last_accessed": "2026-01-28T09:00:00Z",
      "access_count": 12,
      "is_favorite": false
    }
  ],
  
  "preferences": {
    "default_approach": 1,
    "default_top_k": 5,
    "enable_citations": true,
    "enable_thought_chain": false,
    "default_temperature": 0.3,
    "ui_theme": "light"
  },
  
  "groups": ["9f540c2e-...", "3fece663-..."],
  
  "audit": {
    "created_date": "2024-01-15T00:00:00Z",
    "last_login": "2026-02-04T14:30:00Z",
    "total_sessions": 1247,
    "total_queries": 4892
  }
}
```

---

### Container 3: `project_templates` (NEW)

**Purpose**: Store reusable project templates for rapid deployment

**Partition Key**: `/id`

**Schema**:

```json
{
  "id": "template-rag-basic",
  "type": "project_template",
  "partition_key": "template-rag-basic",
  
  "template_metadata": {
    "display_name": "Basic RAG System",
    "description": "Standard retrieval-augmented generation system with hybrid search",
    "category": "rag",
    "version": "1.0.0",
    "created_by": "marco.presta@hrsdc-rhdcc.gc.ca",
    "created_date": "2026-01-15T00:00:00Z",
    "is_public": true,
    "usage_count": 5
  },
  
  "default_config": {
    "cost_attribution": {
      "client": "{{CLIENT_NAME}}",
      "project_code": "{{PROJECT_CODE}}",
      "cost_center": "{{COST_CENTER}}",
      "environment": "dev"
    },
    "system_prompts": {
      "default": "You are an AI assistant. Answer based on the following sources:\n{sources}"
    },
    "feature_flags": {
      "enable_ungrounded_chat": false,
      "enable_math_assistant": false
    },
    "limits": {
      "max_csv_file_size_mb": 20,
      "rate_limit_requests_per_minute": 30
    }
  },
  
  "required_azure_resources": [
    "Azure OpenAI (GPT-4)",
    "Azure Cognitive Search (Basic)",
    "Azure Cosmos DB (Serverless)",
    "Azure Blob Storage (Standard)"
  ],
  
  "deployment_script": "deploy-rag-basic.sh",
  
  "estimated_monthly_cost": {
    "min": 150,
    "max": 300,
    "currency": "USD"
  }
}
```

---

## APIM Policy Updates (Phase 4)

### Inbound Policy with Cost Header Validation & Logging

**File**: APIM policy definition (applied in Phase 4)

```xml
<policies>
  <inbound>
    <base />
    
    <!-- ========================================== -->
    <!-- PHASE 4: Cost Attribution Header Support  -->
    <!-- ========================================== -->
    
    <!-- Validate required authentication headers (existing Phase 4) -->
    <check-header name="x-ms-client-principal-id" failed-check-httpcode="401" failed-check-error-message="Missing user ID" />
    <check-header name="x-ms-client-principal" failed-check-httpcode="401" failed-check-error-message="Missing JWT token" />
    <check-header name="Ocp-Apim-Subscription-Key" failed-check-httpcode="401" failed-check-error-message="Missing API key" />
    
    <!-- Validate required cost attribution headers (NEW) -->
    <check-header name="X-Client" failed-check-httpcode="400" failed-check-error-message="Missing X-Client header" />
    <check-header name="X-Project" failed-check-httpcode="400" failed-check-error-message="Missing X-Project header" />
    <check-header name="X-Environment" failed-check-httpcode="400" failed-check-error-message="Missing X-Environment header" />
    <check-header name="X-CostCenter" failed-check-httpcode="400" failed-check-error-message="Missing X-CostCenter header" />
    
    <!-- Extract cost attribution for logging -->
    <set-variable name="cost_client" value="@(context.Request.Headers.GetValueOrDefault("X-Client", "Unknown"))" />
    <set-variable name="cost_project" value="@(context.Request.Headers.GetValueOrDefault("X-Project", "Unknown"))" />
    <set-variable name="cost_user" value="@(context.Request.Headers.GetValueOrDefault("X-User", "Unknown"))" />
    <set-variable name="cost_environment" value="@(context.Request.Headers.GetValueOrDefault("X-Environment", "dev"))" />
    <set-variable name="cost_phase" value="@(context.Request.Headers.GetValueOrDefault("X-Phase", "Unknown"))" />
    <set-variable name="cost_task" value="@(context.Request.Headers.GetValueOrDefault("X-Task", "Unknown"))" />
    <set-variable name="cost_center" value="@(context.Request.Headers.GetValueOrDefault("X-CostCenter", "Unknown"))" />
    
    <!-- Generate correlation ID if missing -->
    <set-header name="X-Correlation-Id" exists-action="skip">
      <value>@($"req-{DateTime.UtcNow:yyyyMMdd-HHmmss}-{Guid.NewGuid().ToString("N").Substring(0, 8)}")</value>
    </set-header>
    
    <set-variable name="correlation_id" value="@(context.Request.Headers.GetValueOrDefault("X-Correlation-Id", ""))" />
    
    <!-- Log to Application Insights with FinOps dimensions -->
    <log-to-eventhub logger-id="finops-logger" partition-id="0">
      @{
        return new JObject(
          new JProperty("timestamp", DateTime.UtcNow.ToString("o")),
          new JProperty("correlation_id", context.Variables["correlation_id"]),
          new JProperty("endpoint", context.Request.Url.Path),
          new JProperty("method", context.Request.Method),
          new JProperty("user_id", context.Request.Headers.GetValueOrDefault("x-ms-client-principal-id", "anonymous")),
          
          // Cost attribution dimensions
          new JProperty("cost_client", context.Variables["cost_client"]),
          new JProperty("cost_project", context.Variables["cost_project"]),
          new JProperty("cost_user", context.Variables["cost_user"]),
          new JProperty("cost_environment", context.Variables["cost_environment"]),
          new JProperty("cost_phase", context.Variables["cost_phase"]),
          new JProperty("cost_task", context.Variables["cost_task"]),
          new JProperty("cost_center", context.Variables["cost_center"]),
          
          // Performance metrics
          new JProperty("request_size", context.Request.Body.As<string>(preserveContent: true).Length),
          new JProperty("api_version", context.Api.Version)
        ).ToString();
      }
    </log-to-eventhub>
    
    <!-- Extract RBAC groups from JWT (existing Phase 4) -->
    <set-header name="X-User-Groups" exists-action="override">
      <value>@{
        var jwt = context.Request.Headers.GetValueOrDefault("x-ms-client-principal", "");
        if (string.IsNullOrEmpty(jwt)) return "";
        
        var decoded = System.Text.Encoding.UTF8.GetString(Convert.FromBase64String(jwt));
        var claims = Newtonsoft.Json.Linq.JObject.Parse(decoded)["claims"];
        var groups = claims.Where(c => c["typ"].ToString() == "groups")
                           .Select(c => c["val"].ToString());
        return string.Join(",", groups);
      }</value>
    </set-header>
    
    <!-- Calculate role from groups (existing Phase 4) -->
    <set-header name="X-User-Role" exists-action="override">
      <value>@{
        var groups = context.Request.Headers.GetValueOrDefault("X-User-Groups", "").ToLower();
        if (groups.Contains("admin")) return "admin";
        if (groups.Contains("contributor")) return "contributor";
        return "reader";
      }</value>
    </set-header>
    
    <!-- Forward to backend with all headers -->
    <forward-request />
  </inbound>
  
  <backend>
    <base />
  </backend>
  
  <outbound>
    <base />
    
    <!-- Add correlation ID to response -->
    <set-header name="X-Correlation-Id" exists-action="override">
      <value>@((string)context.Variables["correlation_id"])</value>
    </set-header>
    
    <!-- Log response metrics -->
    <log-to-eventhub logger-id="finops-logger" partition-id="0">
      @{
        return new JObject(
          new JProperty("timestamp", DateTime.UtcNow.ToString("o")),
          new JProperty("correlation_id", context.Variables["correlation_id"]),
          new JProperty("status_code", context.Response.StatusCode),
          new JProperty("response_size", context.Response.Body.As<string>(preserveContent: true).Length),
          new JProperty("duration_ms", (DateTime.UtcNow - context.Request.MatchedParameters["timestamp"]).TotalMilliseconds),
          
          // Cost attribution (for chargeback)
          new JProperty("cost_client", context.Variables["cost_client"]),
          new JProperty("cost_project", context.Variables["cost_project"]),
          new JProperty("cost_center", context.Variables["cost_center"])
        ).ToString();
      }
    </log-to-eventhub>
  </outbound>
  
  <on-error>
    <base />
    
    <!-- Log errors with cost attribution -->
    <log-to-eventhub logger-id="finops-logger" partition-id="0">
      @{
        return new JObject(
          new JProperty("timestamp", DateTime.UtcNow.ToString("o")),
          new JProperty("correlation_id", context.Variables["correlation_id"]),
          new JProperty("error_type", "api_error"),
          new JProperty("status_code", context.Response.StatusCode),
          new JProperty("error_message", context.LastError.Message),
          
          // Cost attribution
          new JProperty("cost_client", context.Variables["cost_client"]),
          new JProperty("cost_project", context.Variables["cost_project"]),
          new JProperty("cost_center", context.Variables["cost_center"])
        ).ToString();
      }
    </log-to-eventhub>
  </on-error>
</policies>
```

---

## Backend Implementation (Phase 1-2)

### Configuration Service

**File**: `app/backend/services/config_service.py`

```python
from typing import Dict, Optional
from azure.cosmos import CosmosClient
import os
import logging
import time

LOGGER = logging.getLogger(__name__)

class ConfigurationService:
    """Centralized configuration management from Cosmos DB
    
    Manages project configurations including:
    - Cost attribution metadata
    - System prompts
    - Feature flags
    - Limits and quotas
    - RBAC mappings
    
    Uses 5-minute cache to minimize Cosmos DB RU consumption.
    """
    
    def __init__(self, cosmos_client: CosmosClient):
        self.database = cosmos_client.get_database_client(
            os.getenv("COSMOSDB_USERPROFILE_DATABASE_NAME", "UserInformation")
        )
        self.container = self.database.get_container_client("project_configuration")
        self._cache = {}
        self._cache_expiry = {}
        self._cache_ttl = 300  # 5 minutes
    
    async def get_project_config(self, project_id: str) -> Dict:
        """Get complete project configuration
        
        Args:
            project_id: Project identifier (e.g., "eva-jp-v1.2")
            
        Returns:
            Complete project configuration dictionary
            
        Raises:
            ValueError: If project_id not found and no default config available
        """
        # Check cache
        if project_id in self._cache:
            if time.time() < self._cache_expiry.get(project_id, 0):
                LOGGER.debug(f"[CONFIG] Cache hit for {project_id}")
                return self._cache[project_id]
        
        # Fetch from Cosmos DB
        try:
            query = f"SELECT * FROM c WHERE c.id = @project_id AND c.type = 'project_config'"
            parameters = [{"name": "@project_id", "value": project_id}]
            
            items = list(self.container.query_items(
                query=query,
                parameters=parameters,
                enable_cross_partition_query=True
            ))
            
            if not items:
                LOGGER.warning(f"[CONFIG] No config found for {project_id}, using defaults")
                return self._get_default_config(project_id)
            
            config = items[0]
            
            # Cache
            self._cache[project_id] = config
            self._cache_expiry[project_id] = time.time() + self._cache_ttl
            
            LOGGER.info(f"[CONFIG] Loaded config for {project_id}")
            return config
            
        except Exception as e:
            LOGGER.error(f"[CONFIG] Failed to load config for {project_id}: {e}")
            return self._get_default_config(project_id)
    
    def _get_default_config(self, project_id: str = "default") -> Dict:
        """Fallback default configuration"""
        return {
            "id": project_id,
            "type": "project_config",
            "project_metadata": {
                "display_name": project_id,
                "status": "active"
            },
            "cost_attribution": {
                "client": os.getenv("DEFAULT_CLIENT", "Unknown"),
                "project_code": project_id,
                "cost_center": os.getenv("DEFAULT_COST_CENTER", "Unknown"),
                "environment": os.getenv("ENVIRONMENT", "dev"),
                "phase": "Unknown",
                "default_task": "unknown"
            },
            "system_prompts": {
                "default": "You are a helpful AI assistant."
            },
            "feature_flags": {
                "enable_ungrounded_chat": False,
                "enable_math_assistant": False
            },
            "limits": {
                "max_csv_file_size_mb": 20,
                "rate_limit_requests_per_minute": 30
            },
            "rbac": {
                "enable_rbac": True,
                "default_role": "reader"
            }
        }
    
    async def get_cost_attribution(self, project_id: str) -> Dict:
        """Get cost attribution metadata for API headers
        
        Returns:
            {
                "client": "ESDC-IT",
                "project": "eva-jp-v1.2",
                "cost_center": "AICOE-123",
                "environment": "dev",
                "phase": "Phase2",
                "default_task": "chat-query"
            }
        """
        config = await self.get_project_config(project_id)
        return config.get("cost_attribution", {})
    
    async def get_system_prompt(
        self, 
        project_id: str, 
        prompt_type: str = "default"
    ) -> str:
        """Get system prompt by type
        
        Args:
            project_id: Project identifier
            prompt_type: "default", "ungrounded", "math_assistant", "tabular_data"
            
        Returns:
            System prompt string
        """
        config = await self.get_project_config(project_id)
        prompts = config.get("system_prompts", {})
        return prompts.get(
            prompt_type, 
            prompts.get("default", "You are a helpful AI assistant.")
        )
    
    async def get_feature_flags(self, project_id: str) -> Dict:
        """Get feature flags for project"""
        config = await self.get_project_config(project_id)
        return config.get("feature_flags", {})
    
    async def get_limits(self, project_id: str) -> Dict:
        """Get limits and quotas for project"""
        config = await self.get_project_config(project_id)
        return config.get("limits", {})
    
    async def get_search_config(self, project_id: str) -> Dict:
        """Get search configuration"""
        config = await self.get_project_config(project_id)
        return config.get("search_configuration", {})
    
    def invalidate_cache(self, project_id: Optional[str] = None):
        """Invalidate cache for project (or all projects)
        
        Args:
            project_id: Specific project to invalidate, or None for all
        """
        if project_id:
            if project_id in self._cache:
                del self._cache[project_id]
                del self._cache_expiry[project_id]
                LOGGER.info(f"[CONFIG] Invalidated cache for {project_id}")
        else:
            self._cache.clear()
            self._cache_expiry.clear()
            LOGGER.info("[CONFIG] Invalidated all cache")
```

---

### User Profile Service

**File**: `app/backend/services/user_service.py`

```python
from typing import Dict, List, Optional
from azure.cosmos import CosmosClient
import logging

LOGGER = logging.getLogger(__name__)

class UserService:
    """User profile and project access management"""
    
    def __init__(self, cosmos_client: CosmosClient):
        self.database = cosmos_client.get_database_client(
            os.getenv("COSMOSDB_USERPROFILE_DATABASE_NAME", "UserInformation")
        )
        self.container = self.database.get_container_client("user_profiles")
    
    async def get_user_profile(self, user_id: str) -> Dict:
        """Get user profile including project access
        
        Args:
            user_id: User email (x-ms-client-principal-id)
            
        Returns:
            User profile with available_projects list
        """
        try:
            query = "SELECT * FROM c WHERE c.user_id = @user_id AND c.type = 'user_profile'"
            parameters = [{"name": "@user_id", "value": user_id}]
            
            items = list(self.container.query_items(
                query=query,
                parameters=parameters,
                enable_cross_partition_query=True
            ))
            
            if not items:
                LOGGER.warning(f"[USER] No profile found for {user_id}, creating default")
                return self._create_default_profile(user_id)
            
            return items[0]
            
        except Exception as e:
            LOGGER.error(f"[USER] Failed to load profile for {user_id}: {e}")
            return self._create_default_profile(user_id)
    
    def _create_default_profile(self, user_id: str) -> Dict:
        """Create default user profile"""
        return {
            "id": user_id,
            "type": "user_profile",
            "user_id": user_id,
            "current_project": "eva-jp-v1.2",
            "available_projects": [
                {
                    "project_id": "eva-jp-v1.2",
                    "display_name": "EVA Jurisprudence Assistant",
                    "role": "reader"
                }
            ],
            "preferences": {}
        }
    
    async def get_current_project(self, user_id: str) -> str:
        """Get user's current active project
        
        Returns:
            Project ID (e.g., "eva-jp-v1.2")
        """
        profile = await self.get_user_profile(user_id)
        return profile.get("current_project", "eva-jp-v1.2")
    
    async def switch_project(self, user_id: str, project_id: str) -> bool:
        """Switch user's active project
        
        Args:
            user_id: User email
            project_id: Target project ID
            
        Returns:
            True if successful, False otherwise
        """
        try:
            profile = await self.get_user_profile(user_id)
            
            # Verify user has access to project
            available = profile.get("available_projects", [])
            if not any(p["project_id"] == project_id for p in available):
                LOGGER.warning(f"[USER] {user_id} has no access to {project_id}")
                return False
            
            # Update current project
            profile["current_project"] = project_id
            
            # Update last accessed
            for project in available:
                if project["project_id"] == project_id:
                    from datetime import datetime
                    project["last_accessed"] = datetime.utcnow().isoformat()
                    project["access_count"] = project.get("access_count", 0) + 1
            
            # Save to Cosmos DB
            self.container.upsert_item(profile)
            
            LOGGER.info(f"[USER] {user_id} switched to project {project_id}")
            return True
            
        except Exception as e:
            LOGGER.error(f"[USER] Failed to switch project: {e}")
            return False
    
    async def get_available_projects(self, user_id: str) -> List[Dict]:
        """Get list of projects user has access to"""
        profile = await self.get_user_profile(user_id)
        return profile.get("available_projects", [])
```

---

## API Endpoint Additions (Phase 1-2)

### Backend Endpoints

**File**: `app/backend/app.py` (additions)

```python
from services.config_service import ConfigurationService
from services.user_service import UserService

# Initialize services (after Cosmos DB client initialization)
config_service = ConfigurationService(app.state.cosmosdb_client)
user_service = UserService(app.state.cosmosdb_client)

# ============================================================
# Configuration Endpoints
# ============================================================

@app.get("/api/project-config")
async def get_project_config_endpoint(request: Request):
    """Get current project configuration for frontend
    
    Returns configuration for authenticated user's current project,
    including cost attribution metadata for API headers.
    
    Returns:
        {
            "project_id": "eva-jp-v1.2",
            "display_name": "EVA Jurisprudence Assistant",
            "cost_attribution": {...},
            "feature_flags": {...},
            "limits": {...}
        }
    """
    user_id = request.headers.get("x-ms-client-principal-id")
    if not user_id:
        raise HTTPException(401, "Authentication required")
    
    # Get user's current project
    project_id = await user_service.get_current_project(user_id)
    
    # Load project config
    config = await config_service.get_project_config(project_id)
    
    # Return sanitized config (hide sensitive Azure keys)
    return {
        "project_id": config["id"],
        "display_name": config["project_metadata"]["display_name"],
        "cost_attribution": config["cost_attribution"],
        "feature_flags": config["feature_flags"],
        "limits": config["limits"],
        "ui_customization": config.get("ui_customization", {})
    }

@app.get("/api/user/available-projects")
async def get_available_projects_endpoint(request: Request):
    """Get list of projects user has access to"""
    user_id = request.headers.get("x-ms-client-principal-id")
    if not user_id:
        raise HTTPException(401, "Authentication required")
    
    projects = await user_service.get_available_projects(user_id)
    return {"projects": projects}

@app.post("/api/user/switch-project")
async def switch_project_endpoint(request: Request):
    """Switch user's active project
    
    Request Body:
        {"project_id": "assist-me"}
    """
    user_id = request.headers.get("x-ms-client-principal-id")
    if not user_id:
        raise HTTPException(401, "Authentication required")
    
    data = await request.get_json()
    project_id = data.get("project_id")
    
    if not project_id:
        raise HTTPException(400, "project_id required")
    
    success = await user_service.switch_project(user_id, project_id)
    
    if not success:
        raise HTTPException(403, "No access to project or project not found")
    
    return {"status": "success", "project_id": project_id}

# ============================================================
# Admin Configuration Endpoints (Phase 3)
# ============================================================

@app.get("/api/admin/project-config/{project_id}")
async def get_project_config_admin(request: Request, project_id: str):
    """Get full project configuration (admin only)"""
    # Check admin role
    user_role = request.headers.get("X-User-Role", "reader")
    if user_role != "admin":
        raise HTTPException(403, "Admin access required")
    
    config = await config_service.get_project_config(project_id)
    return config

@app.put("/api/admin/project-config/{project_id}")
async def update_project_config_admin(request: Request, project_id: str):
    """Update project configuration (admin only)"""
    # Check admin role
    user_role = request.headers.get("X-User-Role", "reader")
    if user_role != "admin":
        raise HTTPException(403, "Admin access required")
    
    data = await request.get_json()
    
    # Validate and save to Cosmos DB
    # ... implementation ...
    
    # Invalidate cache
    config_service.invalidate_cache(project_id)
    
    return {"status": "success", "project_id": project_id}
```

---

## Implementation Timeline

### Week 1: Backend Config Infrastructure

**Day 1-2: Data Model & Services**
- [ ] Create `project_configuration` container in Cosmos DB
- [ ] Implement `ConfigurationService` class
- [ ] Implement `UserService` class (enhanced)
- [ ] Seed initial config for `eva-jp-v1.2`

**Day 3-4: API Endpoints**
- [ ] Add `/api/project-config` endpoint
- [ ] Add `/api/user/available-projects` endpoint
- [ ] Add `/api/user/switch-project` endpoint
- [ ] Test with Postman/curl

**Day 5: Integration Testing**
- [ ] Test config loading in chat endpoint
- [ ] Test cost attribution header extraction
- [ ] Test system prompt injection
- [ ] Verify 5-minute cache working

**Deliverables**:
- ✅ Configuration service operational
- ✅ 3 new API endpoints
- ✅ Test evidence collected

---

### Week 2: Frontend Integration

**Day 1-2: Config Service**
- [ ] Create `ConfigService.ts` in frontend
- [ ] Implement config caching in React Context
- [ ] Load config on app initialization

**Day 3-4: API Call Updates**
- [ ] Update `api.ts` to use cost headers from config
- [ ] Remove all hardcoded literals
- [ ] Test all 41 endpoints with dynamic headers

**Day 5: Testing**
- [ ] Smoke test all critical user journeys
- [ ] Verify cost headers in APIM logs
- [ ] Test config reload on project switch

**Deliverables**:
- ✅ Frontend uses dynamic config
- ✅ All API calls include cost headers
- ✅ No hardcoded literals remaining

---

### Week 3: Admin UI

**Day 1-2: Project Config Page**
- [ ] Create `ProjectConfig.tsx` page
- [ ] Implement cost attribution editor
- [ ] Implement system prompt editor
- [ ] Implement feature flag toggles

**Day 3-4: Multi-Project UI**
- [ ] Create `ProjectSwitcher` component
- [ ] Add to main navigation
- [ ] Implement project selection dropdown

**Day 5: Testing**
- [ ] Test config editing (admin role)
- [ ] Test project switching
- [ ] Verify cache invalidation

**Deliverables**:
- ✅ Admin UI operational
- ✅ Project switcher working
- ✅ Config changes reflect immediately

---

### Week 4: APIM Integration

**Day 1-2: APIM Policy Updates**
- [ ] Update inbound policy with cost header validation
- [ ] Add cost header logging to Application Insights
- [ ] Test policy with sample requests

**Day 3-4: End-to-End Testing**
- [ ] Test complete flow: Frontend → APIM → Backend → Cosmos
- [ ] Verify cost headers in APIM logs
- [ ] Verify cost headers in backend logs
- [ ] Verify cost headers in Application Insights

**Day 5: Production Deployment**
- [ ] Deploy updated APIM policies
- [ ] Deploy backend with config service
- [ ] Deploy frontend with dynamic headers
- [ ] Monitor for 24 hours

**Deliverables**:
- ✅ APIM validates cost headers
- ✅ Complete observability chain
- ✅ FinOps chargeback enabled

---

## Success Metrics

### Technical Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Config Load Time** | <100ms | Monitor `/api/project-config` latency |
| **Cache Hit Rate** | >90% | Track cache hits vs misses |
| **API Call Overhead** | <10ms | Measure header extraction time |
| **Config Update Propagation** | <5 seconds | Time from save to cache invalidation |

### Business Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Cost Attribution Coverage** | 100% | All API calls have cost headers |
| **Multi-Project Adoption** | 3+ projects | Count active project configs |
| **Config Changes** | 5+/month | Track config update frequency |
| **Chargeback Accuracy** | 100% | Verify FinOps reports match headers |

---

## Migration Plan (Existing Projects)

### Step 1: Create Project Configs (Week 1)

```python
# scripts/seed_project_configs.py

async def seed_project_config(project_id: str, config_data: Dict):
    """Seed initial project configuration"""
    cosmos_client = CosmosClient(...)
    database = cosmos_client.get_database_client("UserInformation")
    container = database.get_container_client("project_configuration")
    
    config = {
        "id": project_id,
        "type": "project_config",
        "partition_key": project_id,
        **config_data
    }
    
    container.upsert_item(config)
    print(f"[INFO] Seeded config for {project_id}")

# Seed eva-jp-v1.2
await seed_project_config("eva-jp-v1.2", {
    "project_metadata": {...},
    "cost_attribution": {
        "client": "ESDC-IT",
        "project_code": "eva-jp-v1.2",
        "cost_center": "AICOE-123",
        "environment": "dev"
    },
    ...
})

# Seed assist-me
await seed_project_config("assist-me", {...})

# Seed ei-dsst
await seed_project_config("ei-dsst", {...})
```

### Step 2: Update User Profiles (Week 1)

```python
# scripts/migrate_user_profiles.py

async def add_project_access(user_id: str, project_id: str, role: str):
    """Add project to user's available_projects"""
    # ... implementation ...
```

### Step 3: Deploy Backend Changes (Week 2)

- Deploy `ConfigurationService` and `UserService`
- Deploy new API endpoints
- Test with backward compatibility (fallback to env vars)

### Step 4: Deploy Frontend Changes (Week 2)

- Deploy config service
- Deploy updated API calls
- Test with existing users

### Step 5: Monitor & Validate (Week 3)

- Monitor Application Insights for cost headers
- Verify FinOps reports show correct attribution
- Validate chargeback accuracy

---

## Rollback Plan

### If Config Service Fails

**Fallback**: Use environment variables (existing pattern)

```python
# In ConfigurationService._get_default_config()
return {
    "cost_attribution": {
        "client": os.getenv("DEFAULT_CLIENT", "ESDC-IT"),
        "project_code": os.getenv("DEFAULT_PROJECT", "eva-jp-v1.2"),
        "cost_center": os.getenv("DEFAULT_COST_CENTER", "AICOE-123"),
        ...
    }
}
```

**Recovery Time**: Immediate (no code deployment needed)

### If APIM Policy Breaks

**Rollback**: Remove cost header validation temporarily

```xml
<!-- Comment out validation -->
<!--
<check-header name="X-Client" ... />
<check-header name="X-Project" ... />
-->
```

**Recovery Time**: 5 minutes (APIM policy update)

---

## Security Considerations

### Access Control

- **Project Configs**: Only admins can edit via `/api/admin/project-config`
- **User Profiles**: Users can only read own profile
- **Project Switching**: Users can only switch to projects in their `available_projects`

### Data Validation

- **Cost Headers**: APIM validates required headers (400 if missing)
- **Project IDs**: Backend validates against Cosmos DB (404 if not found)
- **Roles**: Backend validates RBAC role before allowing config edits

### Audit Logging

- **Config Changes**: Log all updates with user_id, timestamp, before/after values
- **Project Switches**: Log with correlation ID for FinOps tracking
- **Cost Attribution**: All API calls logged with full cost dimensions

---

## Future Enhancements (Post-Week 4)

### Phase 5: Project Templates (Month 2)

- Create `project_templates` container
- Build template library (RAG, Chatbot, Translation, etc.)
- Add "New Project from Template" UI
- Automate Azure resource provisioning

### Phase 6: Advanced FinOps (Month 3)

- Token usage tracking by project/user
- Cost alerts per project
- Budget enforcement
- Chargeback automation

### Phase 7: Multi-Index Support (Month 4)

- Support multiple search indexes per project
- Index-level cost attribution
- Dynamic index switching in UI

---

## Appendix A: Sample Cosmos DB Queries

### Query 1: Get Project Config

```sql
SELECT * 
FROM c 
WHERE c.id = 'eva-jp-v1.2' 
  AND c.type = 'project_config'
```

### Query 2: Get User's Projects

```sql
SELECT c.available_projects 
FROM c 
WHERE c.user_id = 'marco.presta@hrsdc-rhdcc.gc.ca' 
  AND c.type = 'user_profile'
```

### Query 3: List All Active Projects

```sql
SELECT c.id, c.project_metadata.display_name, c.cost_attribution.cost_center
FROM c 
WHERE c.type = 'project_config' 
  AND c.project_metadata.status = 'active'
ORDER BY c.project_metadata.created_date DESC
```

### Query 4: Find Projects by Cost Center

```sql
SELECT c.id, c.project_metadata.display_name
FROM c 
WHERE c.type = 'project_config' 
  AND c.cost_attribution.cost_center = 'AICOE-123'
```

---

## Appendix B: Power BI Cost Analysis Query

```sql
-- Cost by Project (from Application Insights custom metrics)
SELECT 
  customDimensions.cost_project AS Project,
  customDimensions.cost_client AS Client,
  customDimensions.cost_center AS CostCenter,
  customDimensions.cost_phase AS Phase,
  SUM(CAST(value AS FLOAT)) AS TotalTokens,
  SUM(CAST(value AS FLOAT)) * 0.00003 AS EstimatedCost_USD
FROM customMetrics
WHERE name = 'openai_tokens_used'
  AND timestamp > ago(30d)
GROUP BY 
  customDimensions.cost_project,
  customDimensions.cost_client,
  customDimensions.cost_center,
  customDimensions.cost_phase
ORDER BY EstimatedCost_USD DESC
```

---

**Document Status**: Design Complete - Ready for Implementation  
**Next Step**: Week 1 Day 1 - Create `project_configuration` container in Cosmos DB  
**Owner**: Marco Presta (marco.presta@hrsdc-rhdcc.gc.ca)  
**Related Documents**:
- 07-PHASE3-VALIDATION-REPORT.md (APIM Analysis Complete)
- PHASE3-COMPLETION-REPORT.md (Phase 1-3 Metrics)
- I:\eva-foundation\14-az-finops\FINOPS-OPPORTUNITIES-20260203.md (FinOps Standards)

