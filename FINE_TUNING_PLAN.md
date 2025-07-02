# 🧠 Fine-Tuning Implementation Plan for MDE Threat Triage Agent

## Overview
This plan outlines the implementation of fine-tuning capabilities that will allow the system to learn from threat analysis conversations and improve security detection accuracy over time.

## 🎯 Key Files to Modify/Create

### 1. Database Schema Extensions

**File:** `supabase/migrations/add_fine_tuning_tables.sql` (NEW)
```sql
-- Training Data Collection
CREATE TABLE training_conversations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    chat_id UUID REFERENCES chats(id) ON DELETE CASCADE,
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
    conversation_data JSONB NOT NULL,
    threat_labels JSONB, -- analyst-confirmed threat classifications
    quality_score INTEGER CHECK (quality_score >= 1 AND quality_score <= 5),
    is_approved_for_training BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ
);

-- Fine-tuning Jobs
CREATE TABLE fine_tuning_jobs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    job_name VARCHAR(100) NOT NULL,
    model_type VARCHAR(50) NOT NULL, -- 'threat_detection', 'log_analysis', 'triage_priority'
    base_model VARCHAR(100) NOT NULL,
    training_data_count INTEGER,
    status VARCHAR(20) DEFAULT 'preparing', -- preparing, training, completed, failed
    job_id VARCHAR(100), -- External fine-tuning service job ID
    hyperparameters JSONB,
    metrics JSONB,
    model_file_path TEXT,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMPTZ
);

-- Model Performance Tracking
CREATE TABLE model_performance (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    model_id UUID REFERENCES fine_tuning_jobs(id),
    evaluation_type VARCHAR(50), -- 'accuracy', 'precision', 'recall', 'f1_score'
    score DECIMAL(5,4),
    test_data_size INTEGER,
    evaluation_date TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

### 2. Training Data Collection Components

**File:** `components/fine-tuning/training-data-collector.tsx` (NEW)
```tsx
import { useState, useContext } from 'react'
import { ChatbotUIContext } from '@/context/context'
import { Button } from '@/components/ui/button'
import { Label } from '@/components/ui/label'
import { Textarea } from '@/components/ui/textarea'
import { Select } from '@/components/ui/select'

interface TrainingDataCollectorProps {
  chatId: string
  messages: any[]
}

export const TrainingDataCollector = ({ chatId, messages }: TrainingDataCollectorProps) => {
  const [threatType, setThreatType] = useState('')
  const [severity, setSeverity] = useState('')
  const [analystNotes, setAnalystNotes] = useState('')
  const [qualityScore, setQualityScore] = useState(3)

  const handleSaveTrainingData = async () => {
    // Implementation for saving conversation as training data
  }

  return (
    <div className="space-y-4 p-4 border rounded">
      <h3>Mark for Training Data</h3>
      {/* Threat classification inputs */}
      {/* Quality scoring */}
      {/* Analyst notes */}
      <Button onClick={handleSaveTrainingData}>
        Save as Training Data
      </Button>
    </div>
  )
}
```

**File:** `components/messages/message-actions.tsx` (MODIFY)
```tsx
// Add training data collection button to message actions
const MessageActions = () => {
  // ...existing code...
  
  const handleMarkForTraining = () => {
    // Open training data collection modal
    setShowTrainingDataCollector(true)
  }

  return (
    <div className="flex space-x-2">
      {/* ...existing buttons... */}
      <WithTooltip
        display={<div>Mark for training</div>}
        trigger={
          <IconBrain
            className="cursor-pointer hover:opacity-50"
            size={MESSAGE_ICON_SIZE}
            onClick={handleMarkForTraining}
          />
        }
      />
    </div>
  )
}
```

### 3. API Endpoints for Fine-tuning

**File:** `app/api/fine-tuning/collect-data/route.ts` (NEW)
```typescript
import { NextResponse } from 'next/server'
import { createClient } from '@supabase/supabase-js'
import { getServerProfile } from '@/lib/server/server-chat-helpers'

export async function POST(request: Request) {
  try {
    const { 
      chatId, 
      conversationData, 
      threatLabels, 
      qualityScore, 
      analystNotes 
    } = await request.json()

    const profile = await getServerProfile()
    const supabase = createClient(
      process.env.NEXT_PUBLIC_SUPABASE_URL!,
      process.env.SUPABASE_SERVICE_ROLE_KEY!
    )

    // Store training conversation
    const { data, error } = await supabase
      .from('training_conversations')
      .insert({
        chat_id: chatId,
        user_id: profile.user_id,
        conversation_data: conversationData,
        threat_labels: threatLabels,
        quality_score: qualityScore,
        analyst_notes: analystNotes
      })

    if (error) throw error

    return NextResponse.json({ success: true, data })
  } catch (error) {
    return NextResponse.json({ error: error.message }, { status: 500 })
  }
}
```

**File:** `app/api/fine-tuning/start-job/route.ts` (NEW)
```typescript
import { NextResponse } from 'next/server'
import OpenAI from 'openai'
import { createClient } from '@supabase/supabase-js'

export async function POST(request: Request) {
  try {
    const { modelType, baseModel, trainingDataFilter } = await request.json()

    const supabase = createClient(
      process.env.NEXT_PUBLIC_SUPABASE_URL!,
      process.env.SUPABASE_SERVICE_ROLE_KEY!
    )

    // 1. Collect and format training data
    const trainingData = await prepareTrainingData(trainingDataFilter)
    
    // 2. Upload to OpenAI for fine-tuning
    const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY })
    
    const file = await openai.files.create({
      file: createTrainingFile(trainingData),
      purpose: 'fine-tune'
    })

    // 3. Start fine-tuning job
    const fineTuningJob = await openai.fineTuning.jobs.create({
      training_file: file.id,
      model: baseModel,
      hyperparameters: {
        n_epochs: 3,
        batch_size: 1,
        learning_rate_multiplier: 2
      }
    })

    // 4. Save job info to database
    const { data, error } = await supabase
      .from('fine_tuning_jobs')
      .insert({
        job_name: `${modelType}_${Date.now()}`,
        model_type: modelType,
        base_model: baseModel,
        job_id: fineTuningJob.id,
        status: 'training',
        training_data_count: trainingData.length,
        hyperparameters: fineTuningJob.hyperparameters
      })

    return NextResponse.json({ 
      success: true, 
      jobId: fineTuningJob.id,
      status: fineTuningJob.status 
    })
  } catch (error) {
    return NextResponse.json({ error: error.message }, { status: 500 })
  }
}
```

### 4. Fine-tuning Management Interface

**File:** `app/[locale]/[workspaceid]/fine-tuning/page.tsx` (NEW)
```tsx
'use client'

import { useState, useEffect } from 'react'
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card'
import { Button } from '@/components/ui/button'
import { Badge } from '@/components/ui/badge'
import { Progress } from '@/components/ui/progress'

export default function FineTuningPage() {
  const [jobs, setJobs] = useState([])
  const [trainingData, setTrainingData] = useState([])
  const [selectedJobType, setSelectedJobType] = useState('threat_detection')

  const handleStartFineTuning = async () => {
    // Start fine-tuning process
  }

  return (
    <div className="p-6 space-y-6">
      <h1 className="text-3xl font-bold">Model Fine-tuning</h1>
      
      {/* Training Data Overview */}
      <Card>
        <CardHeader>
          <CardTitle>Training Data Collection</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="grid grid-cols-3 gap-4">
            <div className="text-center">
              <div className="text-2xl font-bold">{trainingData.length}</div>
              <div className="text-sm text-muted-foreground">
                Total Conversations
              </div>
            </div>
            <div className="text-center">
              <div className="text-2xl font-bold">
                {trainingData.filter(d => d.is_approved_for_training).length}
              </div>
              <div className="text-sm text-muted-foreground">
                Approved for Training
              </div>
            </div>
            <div className="text-center">
              <div className="text-2xl font-bold">
                {trainingData.filter(d => d.quality_score >= 4).length}
              </div>
              <div className="text-sm text-muted-foreground">
                High Quality (4+ stars)
              </div>
            </div>
          </div>
        </CardContent>
      </Card>

      {/* Active Jobs */}
      <Card>
        <CardHeader>
          <CardTitle>Fine-tuning Jobs</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="space-y-4">
            {jobs.map(job => (
              <div key={job.id} className="flex items-center justify-between p-4 border rounded">
                <div>
                  <div className="font-semibold">{job.job_name}</div>
                  <div className="text-sm text-muted-foreground">
                    {job.model_type} | {job.base_model}
                  </div>
                </div>
                <div className="flex items-center space-x-2">
                  <Badge variant={job.status === 'completed' ? 'default' : 'secondary'}>
                    {job.status}
                  </Badge>
                  {job.status === 'training' && (
                    <Progress value={job.progress || 0} className="w-24" />
                  )}
                </div>
              </div>
            ))}
          </div>
        </CardContent>
      </Card>

      {/* Start New Fine-tuning */}
      <Card>
        <CardHeader>
          <CardTitle>Start New Fine-tuning</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="space-y-4">
            <div className="grid grid-cols-2 gap-4">
              <Select value={selectedJobType} onValueChange={setSelectedJobType}>
                <SelectTrigger>
                  <SelectValue placeholder="Select model type" />
                </SelectTrigger>
                <SelectContent>
                  <SelectItem value="threat_detection">Threat Detection</SelectItem>
                  <SelectItem value="log_analysis">Log Analysis</SelectItem>
                  <SelectItem value="triage_priority">Triage Priority</SelectItem>
                </SelectContent>
              </Select>
              <Select>
                <SelectTrigger>
                  <SelectValue placeholder="Base model" />
                </SelectTrigger>
                <SelectContent>
                  <SelectItem value="gpt-3.5-turbo">GPT-3.5 Turbo</SelectItem>
                  <SelectItem value="gpt-4">GPT-4</SelectItem>
                </SelectContent>
              </Select>
            </div>
            <Button onClick={handleStartFineTuning} className="w-full">
              Start Fine-tuning Job
            </Button>
          </div>
        </CardContent>
      </Card>
    </div>
  )
}
```

### 5. Training Data Processing Library

**File:** `lib/fine-tuning/data-processor.ts` (NEW)
```typescript
import { Tables } from '@/supabase/types'

export interface TrainingExample {
  messages: Array<{
    role: 'system' | 'user' | 'assistant'
    content: string
  }>
  metadata?: {
    threat_type?: string
    severity?: string
    analyst_confirmed?: boolean
  }
}

export class TrainingDataProcessor {
  static formatConversationForTraining(
    conversation: Tables<'training_conversations'>
  ): TrainingExample {
    const messages = conversation.conversation_data.messages
    const threatLabels = conversation.threat_labels

    return {
      messages: [
        {
          role: 'system',
          content: this.buildSecuritySystemPrompt(threatLabels)
        },
        ...messages.map(msg => ({
          role: msg.role as 'user' | 'assistant',
          content: msg.content
        }))
      ],
      metadata: {
        threat_type: threatLabels?.threat_type,
        severity: threatLabels?.severity,
        analyst_confirmed: conversation.is_approved_for_training
      }
    }
  }

  static buildSecuritySystemPrompt(threatLabels: any): string {
    return `You are a cybersecurity threat analysis expert. 
Your role is to analyze security logs, identify threats, and provide actionable recommendations.

Key capabilities:
- Threat detection and classification
- Log analysis and pattern recognition  
- Risk assessment and prioritization
- Incident response guidance

When analyzing security data:
1. Identify potential threats and IoCs
2. Assess severity and impact
3. Provide clear, actionable recommendations
4. Consider false positive likelihood

${threatLabels ? `
Expected threat classification: ${threatLabels.threat_type}
Expected severity: ${threatLabels.severity}
` : ''}`
  }

  static validateTrainingData(examples: TrainingExample[]): {
    valid: TrainingExample[]
    invalid: TrainingExample[]
    validationErrors: string[]
  } {
    const valid: TrainingExample[] = []
    const invalid: TrainingExample[] = []
    const validationErrors: string[] = []

    examples.forEach((example, index) => {
      const errors = this.validateSingleExample(example, index)
      if (errors.length === 0) {
        valid.push(example)
      } else {
        invalid.push(example)
        validationErrors.push(...errors)
      }
    })

    return { valid, invalid, validationErrors }
  }

  private static validateSingleExample(
    example: TrainingExample, 
    index: number
  ): string[] {
    const errors: string[] = []

    // Check message structure
    if (!example.messages || example.messages.length < 2) {
      errors.push(`Example ${index}: Must have at least 2 messages`)
    }

    // Check for system message
    if (example.messages[0]?.role !== 'system') {
      errors.push(`Example ${index}: First message must be system role`)
    }

    // Check message content length
    example.messages.forEach((msg, msgIndex) => {
      if (!msg.content || msg.content.length < 10) {
        errors.push(`Example ${index}, Message ${msgIndex}: Content too short`)
      }
      if (msg.content.length > 4000) {
        errors.push(`Example ${index}, Message ${msgIndex}: Content too long`)
      }
    })

    return errors
  }
}
```

### 6. Fine-tuned Model Integration

**File:** `app/api/chat/fine-tuned/route.ts` (NEW)
```typescript
import { NextResponse } from 'next/server'
import OpenAI from 'openai'
import { getServerProfile } from '@/lib/server/server-chat-helpers'
import { ChatSettings } from '@/types'

export async function POST(request: Request) {
  try {
    const { chatSettings, messages, fineTunedModelId } = await request.json() as {
      chatSettings: ChatSettings
      messages: any[]
      fineTunedModelId: string
    }

    const profile = await getServerProfile()
    
    const openai = new OpenAI({
      apiKey: profile.openai_api_key || process.env.OPENAI_API_KEY
    })

    // Use the fine-tuned model
    const response = await openai.chat.completions.create({
      model: fineTunedModelId, // e.g., "ft:gpt-3.5-turbo:your-org:threat-detection:abc123"
      messages,
      temperature: chatSettings.temperature,
      max_tokens: 4096,
      stream: true
    })

    const stream = OpenAIStream(response)
    return new StreamingTextResponse(stream)
  } catch (error) {
    return NextResponse.json({ error: error.message }, { status: 500 })
  }
}
```

### 7. Model Selection Enhancement

**File:** `components/models/model-select.tsx` (MODIFY)
```tsx
// Add fine-tuned models to the model selection dropdown
const ModelSelect = () => {
  const [fineTunedModels, setFineTunedModels] = useState([])

  useEffect(() => {
    // Fetch available fine-tuned models
    fetchFineTunedModels()
  }, [])

  const fetchFineTunedModels = async () => {
    const response = await fetch('/api/fine-tuning/models')
    const data = await response.json()
    setFineTunedModels(data.models)
  }

  return (
    <div>
      {/* Existing model options */}
      
      {/* Fine-tuned models section */}
      {fineTunedModels.length > 0 && (
        <div>
          <div className="text-xs font-bold tracking-wide opacity-50 mb-2">
            FINE-TUNED MODELS
          </div>
          {fineTunedModels.map(model => (
            <ModelOption
              key={model.id}
              model={model}
              onSelect={() => onSelectModel(model.id)}
              badge="Fine-tuned"
            />
          ))}
        </div>
      )}
    </div>
  )
}
```

### 8. Performance Monitoring

**File:** `components/fine-tuning/performance-dashboard.tsx` (NEW)
```tsx
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card'
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from 'recharts'

export const PerformanceDashboard = ({ modelId }: { modelId: string }) => {
  const [performanceData, setPerformanceData] = useState([])
  const [currentMetrics, setCurrentMetrics] = useState({
    accuracy: 0,
    precision: 0,
    recall: 0,
    f1_score: 0
  })

  return (
    <div className="space-y-6">
      <h2 className="text-2xl font-bold">Model Performance</h2>
      
      <div className="grid grid-cols-4 gap-4">
        <Card>
          <CardHeader>
            <CardTitle>Accuracy</CardTitle>
          </CardHeader>
          <CardContent>
            <div className="text-3xl font-bold">{(currentMetrics.accuracy * 100).toFixed(1)}%</div>
          </CardContent>
        </Card>
        
        <Card>
          <CardHeader>
            <CardTitle>Precision</CardTitle>
          </CardHeader>
          <CardContent>
            <div className="text-3xl font-bold">{(currentMetrics.precision * 100).toFixed(1)}%</div>
          </CardContent>
        </Card>
        
        <Card>
          <CardHeader>
            <CardTitle>Recall</CardTitle>
          </CardHeader>
          <CardContent>
            <div className="text-3xl font-bold">{(currentMetrics.recall * 100).toFixed(1)}%</div>
          </CardContent>
        </Card>
        
        <Card>
          <CardHeader>
            <CardTitle>F1 Score</CardTitle>
          </CardHeader>
          <CardContent>
            <div className="text-3xl font-bold">{(currentMetrics.f1_score * 100).toFixed(1)}%</div>
          </CardContent>
        </Card>
      </div>

      <Card>
        <CardHeader>
          <CardTitle>Performance Over Time</CardTitle>
        </CardHeader>
        <CardContent>
          <ResponsiveContainer width="100%" height={300}>
            <LineChart data={performanceData}>
              <CartesianGrid strokeDasharray="3 3" />
              <XAxis dataKey="date" />
              <YAxis />
              <Tooltip />
              <Line type="monotone" dataKey="accuracy" stroke="#8884d8" />
              <Line type="monotone" dataKey="precision" stroke="#82ca9d" />
              <Line type="monotone" dataKey="recall" stroke="#ffc658" />
              <Line type="monotone" dataKey="f1_score" stroke="#ff7300" />
            </LineChart>
          </ResponsiveContainer>
        </CardContent>
      </Card>
    </div>
  )
}
```

### 9. Context Enhancement

**File:** `context/context.tsx` (MODIFY)
```tsx
// Add fine-tuning state to the context
interface ChatbotUIContextType {
  // ...existing properties...
  
  // Fine-tuning state
  fineTunedModels: any[]
  setFineTunedModels: (models: any[]) => void
  trainingDataCount: number
  setTrainingDataCount: (count: number) => void
  isCollectingTrainingData: boolean
  setIsCollectingTrainingData: (collecting: boolean) => void
  selectedFineTunedModel: string | null
  setSelectedFineTunedModel: (modelId: string | null) => void
}
```

### 10. Configuration and Environment

**File:** `.env.local` (MODIFY)
```bash
# Existing variables...

# Fine-tuning Configuration
OPENAI_FINE_TUNING_ENABLED=true
FINE_TUNING_MIN_EXAMPLES=50
FINE_TUNING_MAX_EXAMPLES=1000
FINE_TUNING_VALIDATION_SPLIT=0.2

# Model Performance Monitoring
PERFORMANCE_EVALUATION_ENABLED=true
PERFORMANCE_EVALUATION_INTERVAL=86400 # 24 hours in seconds
```

## 🚀 Implementation Steps

### Phase 1: Data Collection (Week 1-2)
1. Create database migrations for training data tables
2. Implement training data collection components
3. Add "Mark for Training" functionality to message actions
4. Create API endpoints for storing training conversations

### Phase 2: Data Processing (Week 3-4)
1. Build training data processor and validator
2. Implement data quality scoring system
3. Create data export functionality for fine-tuning services
4. Add bulk approval/rejection interface for training data

### Phase 3: Fine-tuning Integration (Week 5-6)
1. Implement OpenAI fine-tuning API integration
2. Create job management system
3. Add fine-tuned model selection to UI
4. Implement custom model API endpoint

### Phase 4: Performance Monitoring (Week 7-8)
1. Build performance tracking system
2. Create evaluation metrics dashboard
3. Implement A/B testing between base and fine-tuned models
4. Add automated model performance alerts

### Phase 5: Advanced Features (Week 9-10)
1. Implement continuous learning pipeline
2. Add model versioning and rollback capabilities
3. Create automated retraining triggers
4. Build model comparison and selection tools

## 🎯 Expected Outcomes

### Short-term (1-2 months)
- **Improved Threat Detection**: 15-25% improvement in threat identification accuracy
- **Reduced False Positives**: 20-30% reduction in false positive alerts  
- **Faster Triage**: 40% reduction in time to classify security incidents

### Long-term (3-6 months)
- **Adaptive Learning**: System continuously improves based on analyst feedback
- **Custom Model Library**: Organization-specific models for unique threat patterns
- **Automated Expertise**: AI assistant matches or exceeds junior analyst performance

## 📊 Success Metrics

1. **Model Performance**
   - Accuracy: >90% for threat classification
   - Precision: >85% for high-severity alerts
   - Recall: >95% for critical threats

2. **Training Data Quality**
   - Average quality score: >4.0/5.0
   - Analyst approval rate: >80%
   - Data diversity: Covers all major threat categories

3. **User Adoption**
   - Training data contribution rate: >50% of security conversations
   - Fine-tuned model usage: >70% of threat analysis tasks
   - User satisfaction: >4.5/5.0 rating

This implementation plan provides a comprehensive approach to adding fine-tuning capabilities that will make your threat triage agent increasingly intelligent and effective based on real-world security data and analyst expertise.
