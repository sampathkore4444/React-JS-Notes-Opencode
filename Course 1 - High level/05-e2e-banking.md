# React Mastery Guide — Part 5: E2E Example 2
## Banking Application (Full Architecture)

---

# 1. CORE ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────┐
│                    Banking Application                        │
├─────────────────────────────────────────────────────────────┤
│  security/                                                    │
│    encryption, session management, audit logging              │
├─────────────────────────────────────────────────────────────┤
│  features/                                                    │
│    accounts/  transfers/  bills/  investments/                │
│    statements/  notifications/  support/                      │
├─────────────────────────────────────────────────────────────┤
│  core/                                                        │
│    hooks/  services/  utils/  constants/                      │
├─────────────────────────────────────────────────────────────┤
│  shared/                                                      │
│    ui/  charts/  forms/  layouts/                             │
└─────────────────────────────────────────────────────────────┘
```

Design Considerations:
- **Security-first**: Never expose tokens in URLs, sanitize all inputs, audit all mutations
- **Data integrity**: Double-entry bookkeeping, transaction atomicity
- **Regulatory compliance**: PCI DSS, GDPR, SOX — logs, encryption, data retention
- **Real-time**: Account balance updates, fraud alerts, payment notifications
- **Offline support**: Show cached transactions, queue pending operations

# 2. SECURE API SERVICE

```jsx
// services/bankingApi.js
import api from './api'; // from Part 4
import { encryptSensitive } from '../utils/encryption';

class BankingApi {
  // All financial data is encrypted in transit (HTTPS enforced by api interceptor)

  async getAccounts() {
    const { data } = await api.get('/accounts', {
      headers: {
        'X-Idempotency-Key': crypto.randomUUID(), // prevents duplicate processing
      },
    });
    return data.accounts.map(account => ({
      ...account,
      // Mask account numbers for display
      displayNumber: this.maskNumber(account.number),
    }));
  }

  async getTransactions(accountId, { page = 1, limit = 50, fromDate, toDate, type } = {}) {
    const params = new URLSearchParams({
      page, limit, accountId,
      ...(fromDate && { from: fromDate }),
      ...(toDate && { to: toDate }),
      ...(type && { type }),
    });

    const { data } = await api.get(`/transactions?${params}`);
    return {
      transactions: data.transactions,
      pagination: {
        page: data.page,
        totalPages: data.totalPages,
        total: data.total,
        hasMore: data.page < data.totalPages,
      },
    };
  }

  async initiateTransfer(transferData) {
    // Idempotency prevents duplicate transfers on retry
    const idempotencyKey = crypto.randomUUID();

    const { data } = await api.post('/transfers', {
      ...transferData,
      idempotencyKey,
      // Encrypt sensitive data before sending
      // (HTTPS already encrypts, but this adds application-level encryption)
      memo: encryptSensitive(transferData.memo),
    });

    return data;
  }

  async getBalance(accountId) {
    const { data } = await api.get(`/accounts/${accountId}/balance`);
    return data.balance;
  }

  async getStatements(accountId, year, month) {
    const { data } = await api.get(`/accounts/${accountId}/statements/${year}/${month}`);
    return data;
  }

  maskNumber(number) {
    return `****${number.slice(-4)}`;
  }
}

export const bankingApi = new BankingApi();
```

# 3. ACCOUNT DASHBOARD

```jsx
// features/accounts/hooks/useAccountDashboard.js
import { useState, useEffect, useCallback, useMemo } from 'react';
import { bankingApi } from '../../../services/bankingApi';

export const useAccountDashboard = () => {
  const [accounts, setAccounts] = useState([]);
  const [selectedAccountId, setSelectedAccountId] = useState(null);
  const [transactions, setTransactions] = useState([]);
  const [pagination, setPagination] = useState({ page: 1, hasMore: false });
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const [filters, setFilters] = useState({ fromDate: '', toDate: '', type: '' });

  const selectedAccount = useMemo(() =>
    accounts.find(a => a.id === selectedAccountId) || accounts[0],
    [accounts, selectedAccountId]
  );

  // Fetch accounts on mount
  useEffect(() => {
    let cancelled = false;
    const fetchAccounts = async () => {
      try {
        const data = await bankingApi.getAccounts();
        if (!cancelled) {
          setAccounts(data);
          if (!selectedAccountId && data.length > 0) {
            setSelectedAccountId(data[0].id);
          }
        }
      } catch (err) {
        if (!cancelled) setError(err.message);
      } finally {
        if (!cancelled) setLoading(false);
      }
    };
    fetchAccounts();
    return () => { cancelled = true; };
  }, [selectedAccountId]);

  // Fetch transactions when account or filters change
  useEffect(() => {
    if (!selectedAccountId) return;

    let cancelled = false;
    const fetchTransactions = async () => {
      setLoading(true);
      try {
        const data = await bankingApi.getTransactions(selectedAccountId, {
          ...filters,
          page: 1,
        });
        if (!cancelled) {
          setTransactions(data.transactions);
          setPagination(data.pagination);
        }
      } catch (err) {
        if (!cancelled) setError(err.message);
      } finally {
        if (!cancelled) setLoading(false);
      }
    };
    fetchTransactions();
    return () => { cancelled = true; };
  }, [selectedAccountId, filters]);

  const loadMore = useCallback(async () => {
    if (!pagination.hasMore || loading) return;

    const nextPage = pagination.page + 1;
    const data = await bankingApi.getTransactions(selectedAccountId, {
      ...filters,
      page: nextPage,
    });
    setTransactions(prev => [...prev, ...data.transactions]);
    setPagination(data.pagination);
  }, [pagination, loading, selectedAccountId, filters]);

  // Real-time balance updates via WebSocket
  useEffect(() => {
    if (!selectedAccountId) return;

    const ws = new WebSocket(`wss://api.bank.com/accounts/${selectedAccountId}/updates`);
    ws.onmessage = (event) => {
      const update = JSON.parse(event.data);
      if (update.type === 'balance_change') {
        setAccounts(prev => prev.map(acc =>
          acc.id === selectedAccountId
            ? { ...acc, balance: update.newBalance }
            : acc
        ));
        // Prepend new transaction
        setTransactions(prev => [update.transaction, ...prev]);
      }
    };
    return () => ws.close();
  }, [selectedAccountId]);

  return {
    accounts, selectedAccount, setSelectedAccountId,
    transactions, loading, error, loadMore, pagination,
    filters, setFilters,
  };
};
```

# 4. TRANSFER FORM (with validation)

```jsx
// features/transfers/components/TransferForm.jsx
import { useState, useCallback, useReducer } from 'react';
import { bankingApi } from '../../../services/bankingApi';
import { validateRoutingNumber, validateAccountNumber } from '../utils/validation';
import { useTransfer } from '../hooks/useTransfer';

const TRANSFER_TYPES = {
  INTERNAL: 'internal',      // Same bank
  EXTERNAL: 'external',      // Other bank (ACH)
  WIRE: 'wire',              // Wire transfer
  INTERNATIONAL: 'international',
};

const initialState = {
  fromAccount: '',
  toAccount: '',
  toRoutingNumber: '',
  amount: '',
  type: 'internal',
  memo: '',
  frequency: 'once', // once, weekly, monthly
  scheduledDate: '',
  errors: {},
};

function transferReducer(state, action) {
  switch (action.type) {
    case 'SET_FIELD':
      return { ...state, [action.payload.field]: action.payload.value, errors: {} };
    case 'SET_ERRORS':
      return { ...state, errors: action.payload };
    case 'RESET':
      return initialState;
    default:
      return state;
  }
}

export const TransferForm = ({ accounts, onSuccess }) => {
  const [state, dispatch] = useReducer(transferReducer, initialState);
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [serverError, setServerError] = useState(null);
  const [confirmation, setConfirmation] = useState(null);

  // real-time balance check
  const fromAccount = accounts.find(a => a.id === state.fromAccount);
  const exceedsBalance = fromAccount && parseFloat(state.amount) > fromAccount.balance;
  const dailyLimit = 10000;
  const exceedsLimit = parseFloat(state.amount) > dailyLimit;

  const validate = useCallback(() => {
    const errors = {};

    if (!state.fromAccount) errors.fromAccount = 'Select source account';
    if (!state.amount || parseFloat(state.amount) <= 0) errors.amount = 'Enter valid amount';
    if (parseFloat(state.amount) > 100000) errors.amount = 'Amount exceeds $100,000 limit';
    if (exceedsBalance) errors.amount = 'Insufficient funds';

    if (state.type === 'internal') {
      if (!state.toAccount) errors.toAccount = 'Select destination account';
      if (state.toAccount === state.fromAccount) errors.toAccount = 'Cannot transfer to same account';
    } else {
      if (!state.toAccount) errors.toAccount = 'Enter account number';
      else if (!validateAccountNumber(state.toAccount)) errors.toAccount = 'Invalid account number';
      if (state.type !== 'international' && !state.toRoutingNumber) {
        errors.toRoutingNumber = 'Enter routing number';
      } else if (state.toRoutingNumber && !validateRoutingNumber(state.toRoutingNumber)) {
        errors.toRoutingNumber = 'Invalid routing number';
      }
    }

    if (state.type === 'international' && !state.iban) errors.iban = 'Enter IBAN';
    if (state.frequency !== 'once' && !state.scheduledDate) errors.scheduledDate = 'Select date';

    return errors;
  }, [state, exceedsBalance]);

  const handleSubmit = useCallback(async (e) => {
    e.preventDefault();
    const errors = validate();
    if (Object.keys(errors).length > 0) {
      dispatch({ type: 'SET_ERRORS', payload: errors });
      return;
    }

    // Show confirmation screen first
    setConfirmation(state);
  }, [state, validate]);

  const confirmTransfer = useCallback(async () => {
    setIsSubmitting(true);
    setServerError(null);

    try {
      const result = await bankingApi.initiateTransfer({
        fromAccountId: confirmation.fromAccount,
        toAccountNumber: confirmation.toAccount,
        amount: parseFloat(confirmation.amount),
        type: confirmation.type,
        memo: confirmation.memo,
        frequency: confirmation.frequency === 'once' ? null : confirmation.frequency,
        scheduledDate: confirmation.scheduledDate || null,
      });

      onSuccess?.(result);
    } catch (err) {
      setServerError(err.response?.data?.message || 'Transfer failed. Please try again.');
    } finally {
      setIsSubmitting(false);
    }
  }, [confirmation, onSuccess]);

  if (confirmation) {
    return (
      <div className="transfer-confirmation">
        <h2>Review Transfer</h2>
        <div className="confirmation-details">
          <div>
            <strong>From:</strong> {fromAccount?.name} ({fromAccount?.displayNumber})
          </div>
          <div><strong>To:</strong> {confirmation.toAccount}</div>
          <div><strong>Amount:</strong> ${parseFloat(confirmation.amount).toFixed(2)}</div>
          <div><strong>Type:</strong> {confirmation.type}</div>
          <div><strong>Memo:</strong> {confirmation.memo || '—'}</div>
          <div><strong>Frequency:</strong> {confirmation.frequency}</div>
        </div>
        {serverError && <Alert type="error">{serverError}</Alert>}
        <div className="confirmation-actions">
          <button onClick={() => setConfirmation(null)} disabled={isSubmitting}>
            Edit
          </button>
          <button onClick={confirmTransfer} disabled={isSubmitting}>
            {isSubmitting ? 'Processing...' : 'Confirm Transfer'}
          </button>
        </div>
      </div>
    );
  }

  return (
    <form onSubmit={handleSubmit} className="transfer-form" noValidate>
      <div className="form-group">
        <label>From Account</label>
        <select
          value={state.fromAccount}
          onChange={e => dispatch({ type: 'SET_FIELD', payload: { field: 'fromAccount', value: e.target.value } })}
        >
          <option value="">Select account</option>
          {accounts.map(a => (
            <option key={a.id} value={a.id}>
              {a.name} — ${a.balance.toFixed(2)}
            </option>
          ))}
        </select>
        {state.errors.fromAccount && <ErrorMsg>{state.errors.fromAccount}</ErrorMsg>}
      </div>

      <div className="form-group">
        <label>Transfer Type</label>
        <div className="transfer-types">
          {Object.entries(TRANSFER_TYPES).map(([key, value]) => (
            <button
              key={value}
              type="button"
              className={`type-btn ${state.type === value ? 'active' : ''}`}
              onClick={() => dispatch({ type: 'SET_FIELD', payload: { field: 'type', value } })}
            >
              {key.charAt(0) + key.slice(1).toLowerCase()}
            </button>
          ))}
        </div>
      </div>

      <div className="form-group">
        <label>Amount</label>
        <div className="amount-input">
          <span className="currency">$</span>
          <input
            type="number"
            step="0.01"
            min="0.01"
            value={state.amount}
            onChange={e => dispatch({ type: 'SET_FIELD', payload: { field: 'amount', value: e.target.value } })}
            placeholder="0.00"
          />
        </div>
        {exceedsBalance && <Warning>Insufficient funds. Available: ${fromAccount?.balance.toFixed(2)}</Warning>}
        {exceedsLimit && <Warning>Daily transfer limit: $10,000</Warning>}
        {state.errors.amount && <ErrorMsg>{state.errors.amount}</ErrorMsg>}
      </div>

      {state.type === 'internal' ? (
        <div className="form-group">
          <label>To Account</label>
          <select
            value={state.toAccount}
            onChange={e => dispatch({ type: 'SET_FIELD', payload: { field: 'toAccount', value: e.target.value } })}
          >
            <option value="">Select destination</option>
            {accounts.filter(a => a.id !== state.fromAccount).map(a => (
              <option key={a.id} value={a.id}>{a.name} ({a.displayNumber})</option>
            ))}
          </select>
          {state.errors.toAccount && <ErrorMsg>{state.errors.toAccount}</ErrorMsg>}
        </div>
      ) : (
        <>
          <div className="form-group">
            <label>Account Number</label>
            <input
              value={state.toAccount}
              onChange={e => dispatch({ type: 'SET_FIELD', payload: { field: 'toAccount', value: e.target.value } })}
              placeholder="123456789"
              maxLength={17}
            />
            {state.errors.toAccount && <ErrorMsg>{state.errors.toAccount}</ErrorMsg>}
          </div>

          {state.type !== 'international' && (
            <div className="form-group">
              <label>Routing Number</label>
              <input
                value={state.toRoutingNumber}
                onChange={e => dispatch({ type: 'SET_FIELD', payload: { field: 'toRoutingNumber', value: e.target.value } })}
                placeholder="021000021"
                maxLength={9}
              />
              {state.errors.toRoutingNumber && <ErrorMsg>{state.errors.toRoutingNumber}</ErrorMsg>}
            </div>
          )}
        </>
      )}

      <div className="form-group">
        <label>Memo (optional)</label>
        <input
          value={state.memo}
          onChange={e => dispatch({ type: 'SET_FIELD', payload: { field: 'memo', value: e.target.value } })}
          placeholder="What's this for?"
          maxLength={100}
        />
      </div>

      <div className="form-row">
        <div className="form-group">
          <label>Frequency</label>
          <select
            value={state.frequency}
            onChange={e => dispatch({ type: 'SET_FIELD', payload: { field: 'frequency', value: e.target.value } })}
          >
            <option value="once">One time</option>
            <option value="weekly">Weekly</option>
            <option value="monthly">Monthly</option>
          </select>
        </div>

        {state.frequency !== 'once' && (
          <div className="form-group">
            <label>Start Date</label>
            <input
              type="date"
              value={state.scheduledDate}
              onChange={e => dispatch({ type: 'SET_FIELD', payload: { field: 'scheduledDate', value: e.target.value } })}
              min={new Date().toISOString().split('T')[0]}
            />
            {state.errors.scheduledDate && <ErrorMsg>{state.errors.scheduledDate}</ErrorMsg>}
          </div>
        )}
      </div>

      <button type="submit" className="btn-primary btn-block">
        Review Transfer
      </button>
    </form>
  );
};
```

# 5. TRANSACTION HISTORY WITH INFINITE SCROLL

```jsx
// features/accounts/components/TransactionList.jsx
import { useCallback, useRef, useEffect } from 'react';
import { formatCurrency, formatDate } from '../../../utils/format';

const CATEGORY_ICONS = {
  salary: '💰', food: '🍔', transport: '🚗',
  shopping: '🛍️', bills: '📄', entertainment: '🎬',
  health: '🏥', transfer: '🔄', other: '📝',
};

const TransactionRow = memo(({ transaction, isPending }) => (
  <div className={`transaction-row ${isPending ? 'pending' : ''}`}>
    <div className="transaction-icon">
      {CATEGORY_ICONS[transaction.category] || '📝'}
    </div>
    <div className="transaction-info">
      <div className="transaction-description">{transaction.description}</div>
      <div className="transaction-meta">
        <span>{formatDate(transaction.date)}</span>
        {transaction.isPending && <span className="badge-pending">Pending</span>}
        {transaction.category && <span className="badge-category">{transaction.category}</span>}
      </div>
    </div>
    <div className={`transaction-amount ${transaction.type}`}>
      {transaction.type === 'debit' ? '-' : '+'}${formatCurrency(transaction.amount)}
    </div>
  </div>
));

export const TransactionList = ({ transactions, onLoadMore, hasMore, loading }) => {
  const observerRef = useRef(null);
  const loadMoreRef = useCallback(node => {
    if (observerRef.current) observerRef.current.disconnect();
    observerRef.current = new IntersectionObserver(entries => {
      if (entries[0].isIntersecting && hasMore && !loading) {
        onLoadMore();
      }
    }, { threshold: 0.1 });
    if (node) observerRef.current.observe(node);
  }, [hasMore, loading, onLoadMore]);

  return (
    <div className="transaction-list">
      {transactions.length === 0 && !loading ? (
        <EmptyState icon="📭" message="No transactions found" />
      ) : (
        <>
          {transactions.map((tx, index) => (
            <TransactionRow
              key={tx.id}
              transaction={tx}
              isPending={tx.status === 'pending'}
            />
          ))}

          {/* Sentry element for intersection observer */}
          <div ref={loadMoreRef} className="load-more-sentry">
            {loading && <Spinner size="small" />}
            {!hasMore && transactions.length > 0 && (
              <p className="end-of-list">All transactions loaded</p>
            )}
          </div>
        </>
      )}
    </div>
  );
};
```

# 6. STATEMENT DOWNLOAD

```jsx
// features/statements/components/StatementDownloader.jsx
import { useState, useCallback } from 'react';
import { bankingApi } from '../../../services/bankingApi';

const FORMATS = [
  { value: 'pdf', label: 'PDF', icon: '📄' },
  { value: 'csv', label: 'CSV', icon: '📊' },
  { value: 'ofx', label: 'OFX (QuickBooks)', icon: '💼' },
];

export const StatementDownloader = ({ accountId }) => {
  const [month, setMonth] = useState(new Date().getMonth());
  const [year, setYear] = useState(new Date().getFullYear());
  const [format, setFormat] = useState('pdf');
  const [downloading, setDownloading] = useState(false);
  const [error, setError] = useState(null);

  const availableYears = Array.from({ length: 3 }, (_, i) => new Date().getFullYear() - i);

  const handleDownload = useCallback(async () => {
    setDownloading(true);
    setError(null);
    try {
      const blob = await bankingApi.getStatement(accountId, year, month + 1, format);
      const url = window.URL.createObjectURL(blob);
      const link = document.createElement('a');
      link.href = url;
      link.download = `statement-${year}-${String(month + 1).padStart(2, '0')}.${format}`;
      link.click();
      window.URL.revokeObjectURL(url);
    } catch (err) {
      setError('Failed to generate statement. Please try again.');
    } finally {
      setDownloading(false);
    }
  }, [accountId, year, month, format]);

  return (
    <div className="statement-downloader">
      <h2>Download Statement</h2>

      {error && <Alert type="error">{error}</Alert>}

      <div className="statement-controls">
        <div className="form-group">
          <label>Month</label>
          <select value={month} onChange={e => setMonth(Number(e.target.value))}>
            {Array.from({ length: 12 }, (_, i) => (
              <option key={i} value={i}>
                {new Date(0, i).toLocaleString('en', { month: 'long' })}
              </option>
            ))}
          </select>
        </div>

        <div className="form-group">
          <label>Year</label>
          <select value={year} onChange={e => setYear(Number(e.target.value))}>
            {availableYears.map(y => <option key={y} value={y}>{y}</option>)}
          </select>
        </div>

        <div className="form-group">
          <label>Format</label>
          <div className="format-options">
            {FORMATS.map(f => (
              <button
                key={f.value}
                type="button"
                className={`format-btn ${format === f.value ? 'active' : ''}`}
                onClick={() => setFormat(f.value)}
              >
                {f.icon} {f.label}
              </button>
            ))}
          </div>
        </div>
      </div>

      <button
        onClick={handleDownload}
        disabled={downloading}
        className="btn-primary btn-block"
      >
        {downloading ? 'Generating...' : 'Download Statement'}
      </button>
    </div>
  );
};
```

# 7. AUDIT LOG (for compliance)

```jsx
// hooks/useAuditLog.js — Logs all sensitive operations
export const useAuditLog = () => {
  const logEvent = useCallback((action, details) => {
    const auditEntry = {
      timestamp: new Date().toISOString(),
      action,
      details: {
        ...details,
        // Never log raw passwords, card numbers, or SSNs
        ip: null, // Server should capture this
        userAgent: navigator.userAgent,
      },
      // Client-side: send to server for permanent logging
    };

    // Send to server in background (fire and forget)
    navigator.sendBeacon('/api/audit', JSON.stringify(auditEntry));

    // Also log to local for debugging (never in production!)
    if (import.meta.env.DEV) {
      console.log('[AUDIT]', action, details);
    }
  }, []);

  return { logEvent };
};

// Usage in TransferForm
const { logEvent } = useAuditLog();

const confirmTransfer = async () => {
  logEvent('transfer_initiated', {
    fromAccount: maskAccount(confirmation.fromAccount),
    toAccount: maskAccount(confirmation.toAccount),
    amount: parseFloat(confirmation.amount),
    type: confirmation.type,
    // No PII in audit logs!
  });
  // ... proceed with transfer
};
```

# INTERVIEW TIPS FOR BANKING

1. **Security is job #1** — Mask sensitive data, use idempotency keys, never log credentials
2. **Data integrity** — All financial operations should be atomic. Use reducers for guaranteed state transitions
3. **Real-time updates** — WebSocket for balance changes, but always fall back to polling
4. **Regulatory compliance** — Audit all mutations, provide statement downloads in multiple formats
5. **Error handling** — Network failures during transfer must show clear recovery instructions
6. **Session management** — Auto-logout after inactivity, refresh token rotation
7. **Performance** — Transaction history can be huge — virtualize or paginate

---

# 5 INTERVIEW QUESTIONS (Banking Domain)

**Q1:** How do you prevent duplicate transfers if the user double-clicks submit?
**A:** Disable button immediately on click, use idempotency key per request, show loading state, and implement server-side deduplication. The idempotency key ensures even if the request is sent twice, it only processes once.

**Q2:** How would you handle real-time balance updates across multiple open tabs?
**A:** Use BroadcastChannel API for same-origin tabs: when one tab receives a WebSocket update, broadcast to all tabs. Fallback: use localStorage event listener. This ensures all tabs show consistent balance.

**Q3:** How do you safely display account numbers?
**A:** Mask all but last 4 digits. Never store full numbers in state, Redux, or logs. Only display full number on a dedicated "Show Details" action that auto-hides after timeout.

**Q4:** Design a transaction search that's fast even with 100k+ transactions.
**A:** Use Web Workers for client-side filtering. Implement server-side search with debounced API calls. Virtualize results (react-window). Cache recent searches. Show partial results while fetching more.

**Q5:** How would you implement a fraud detection indicator on the frontend?
**A:** Monitor for unusual patterns: rapid successive transfers, large amounts relative to history, new payees. Show friction (2FA, confirmation dialog, cooling period) based on risk score from backend. Display contextual warnings.
