# 🛡️ Security, Rate-Limiting & Billing Plans

To protect clients and the system from run-away hosting/API costs, `chatbotRAG` enforces strict security safeguards, usage counters, and rate limit protections based on the organization's tier.

---

## 📊 Subscription Plans & Resource Limits

| Feature Limit | Starter Plan | Growth Plan | Pro Plan (Future) |
| :--- | :--- | :--- | :--- |
| **Monthly Cost** | Free / $9 | $29 / month | $99 / month |
| **Document Uploads** | Max 5 files | Max 50 files | Unlimited |
| **Max Individual File Size** | 2 MB | 5 MB | 20 MB (multimodal) |
| **Monthly Gemini Token Limit**| 50,000 tokens | 500,000 tokens | 5,000,000 tokens |
| **Daily Message Limit (All users)**| 200 messages | 2,000 messages | 10,000 messages |
| **Spam Throttling (Per User)** | 5 msgs / minute | 15 msgs / minute | 60 msgs / minute |

---

## 🛑 Rate-Limiting & Spam Throttling

Spam protection is implemented at the database level using a sliding-window counter. When a message arrives at the webhook:
1. The Edge Function runs a check to ensure the organization's monthly token limit has not been exceeded.
2. The Edge Function runs an RPC query to verify the telegram user has not exceeded their per-user rate limit (e.g., 5 messages per minute for Starter).

### Database Rate-Limit Check Function
```sql
create or replace function public.check_user_rate_limit(
  p_org_id uuid,
  p_telegram_chat_id bigint,
  p_limit_window_seconds int,
  p_max_allowed int
)
returns boolean
language plpgsql
as $$
declare
  v_message_count int;
begin
  -- Count messages in the sliding time window
  select count(*)
  into v_message_count
  from public.usage_logs
  where org_id = p_org_id
    and telegram_chat_id = p_telegram_chat_id
    and created_at >= (now() - (p_limit_window_seconds || ' seconds')::interval);

  if v_message_count >= p_max_allowed then
    return false; -- Rate limit exceeded
  else
    return true;  -- Allow query
  end if;
end;
$$;
```

If the rate limit check returns `false`, the system short-circuits, bypasses the Gemini API, and responds immediately to the user:
> ⚠️ **Limit Warning:** *"You are sending queries too quickly. Please wait a minute before trying again."*

---

## 📈 Real-Time Token & Usage Monitoring

Gemini API returns accurate token consumption fields inside the metadata block of its response payload:
```json
{
  "usageMetadata": {
    "promptTokenCount": 240,
    "candidatesTokenCount": 85,
    "totalTokenCount": 325
  }
}
```

### Logging Consumption
Every valid RAG transaction writes to `usage_logs`:
1. Record `tokens_consumed` as the sum of prompt and response tokens.
2. Increment the organization's aggregated monthly token counter.
3. If an organization exceeds **90%** of their monthly quota, send a notification banner in the React dashboard.
4. If an organization reaches **100%** of their quota, subsequent bot interactions return:
   > 🚫 *"This bot has temporarily exceeded its monthly usage limits. Please contact the administrator to upgrade their plan."*
