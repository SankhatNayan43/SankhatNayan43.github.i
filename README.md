Cracking the Code: How We Built an AI Interview Mocker and Monitored it with SigNoz
Preparing for technical job interviews is stressful. Static Q&A lists don't cut it, and scheduling mock interviews with human mentors is expensive and hard to coordinate. To solve this, our team built AI-Interview-Mocker—an intelligent, full-stack application that acts as an adaptive, 24/7 mock interviewer.

But building a complex application driven by real-time LLM requests and dynamic audio/video evaluation presents a major engineering challenge: latency and reliability. To make sure our AI interviewer felt responsive and smooth, we plugged in SigNoz for complete end-to-end observability.

Here is a breakdown of what we built, what we learned, and how we tamed our distributed architecture.

🏗️ What We Built: The Architecture
AI-Interview-Mocker is an intelligent platform that parses a user's target job role, experience level, or job description to dynamically generate highly relevant, challenging technical questions.

Key Features
Personalized AI Panels: Generates 5+ highly customized questions on-demand.

Voice & Text Interactions: Candidates can respond via text or mic input to capture speech parameters.

Smart Feedback Engine: Delivers immediate grading on technical accuracy, grammar, clarity, and overall confidence.

Comprehensive Dashboard: Tracks session history, performance metrics, and progress curves over time.

The Tech Stack
We built our application using a modern, scalable stack:

Frontend: Next.js, TailwindCSS, and ShadCN components for an ultra-responsive user interface.

Authentication: Clerk for seamless user session management.

Database & ORM: PostgreSQL paired with Drizzle ORM.

AI Core Engine: Google Gemini API orchestrated alongside custom prompt templates.

Observability: OpenTelemetry (OTel) instrumentation piped directly into SigNoz.

Our system functions via a microservice-oriented gateway strategy. When a user kicks off an interview session, the application communicates with Clerk for verification, fetches configuration settings from PostgreSQL via Drizzle, handles dynamic stream generation with the Gemini API, and writes telemetry logs out to SigNoz.

🔬 Bringing Visibility out of Chaos: Instrumenting SigNoz
With multi-second downstream AI inference times and nested database queries, we could not afford to blindly guess why a request was taking too long. We integrated SigNoz using the official OpenTelemetry SDKs to get absolute clarity over our execution paths.

1. Manual Span Customization for LLM Tracking
While auto-instrumentation tracked our raw HTTP requests, the real performance variance lay within the Gemini API calls. We wrapped our core generation function in a custom trace span:

JavaScript
import { trace } from "@opentelemetry/api";

const tracer = trace.getTracer("ai-interview-service");

async function generateInterviewQuestion(jobRole, seniority) {
  return t.withSpan(tracer.startSpan("GeminiAPI_GenerateQuestion"), async (span) => {
    try {
      span.setAttribute("job.role", jobRole);
      span.setAttribute("job.seniority", seniority);
      
      const response = await geminiEngine.prompt({
        text: `Generate an interview question for a ${seniority} ${jobRole}...`
      });
      
      span.setStatus({ code: SpanStatusCode.OK });
      return response;
    } catch (error) {
      span.recordException(error);
      span.setStatus({ code: SpanStatusCode.ERROR, message: error.message });
      throw error;
    } finally {
      span.end();
    }
  });
}
2. Custom Dashboards inside SigNoz
We built a specialized telemetry view in SigNoz to isolate application metrics from infrastructure anomalies. We tracked three primary indicators:

Inference P99 Latency: Tracking precisely how long the Gemini API takes to spit out evaluation cards.

Drizzle Transaction Durations: Assuring our PostgreSQL reads and writes for history logging stayed under 50ms.

Clerk Auth Overhead: Isolating whether handshake latency was originating from our code or external provider APIs.

🌋 The Challenges We Solved (Driven by Observability)
Challenge 1: The 4-Second Dashboard Lag
The Symptom: When users loaded their main dashboard, the page stalled for over four seconds before rendering their interview history.

The SigNoz Discovery: Looking at the distributed tracing timeline view, we noticed a massive cascading block of database queries. Drizzle ORM was executing "N+1 queries"—running an individual SQL fetch for every single historical answer score instead of performing a single unified JOIN query across the session tables.

The Fix: We rewrote the fetching layer into a clean, relational query block that aggregates the session summaries in one database trip. SigNoz confirmed that dashboard load times dropped immediately from 4200ms down to 180ms.

Challenge 2: Gracefully Handling Rate Limits
The Symptom: Random intermittent errors during intensive response evaluation rounds.

The SigNoz Discovery: By setting up an alert in SigNoz for HTTP status code errors, we tracked down a spike of 429 Too Many Requests coming directly from our external AI provider under moderate parallel traffic loads.

The Fix: Using the exception traces captured by SigNoz, we added an exponential backoff retry mechanism inside our API routing block. This handled the traffic spikes gracefully without crashing the user's active interview session.

🧠 What We Learnt
Building this project taught our team invaluable engineering lessons:

AI Performance is Unpredictable: Unlike traditional databases, AI inference times are highly variable. Designing systems with asynchronous states and real-time observability from day one is mandatory.

OpenTelemetry is a Superpower: Standardizing our metrics and logs via OTel meant we could confidently run diagnostics across both frontend transitions and backend API hooks uniformly.

Observability Saves Development Time: Instead of peppering console.log statements everywhere and redeploying code blindly, SigNoz allowed us to track the root causes of architectural failures in production within seconds.

Check out our codebase!
💻 View the source on GitHub: SankhatNayan43 / AI-Interview-Mocker
