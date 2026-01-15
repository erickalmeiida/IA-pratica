// Serviço de colaboração: salva versões rapidamente e valida em background
class CollabService {
  constructor(validator) {
    // validator: função async(content) => { valid: boolean, errors?: string[] }
    this.validator = validator || (async () => ({ valid: true }));
    // armazenamento em memória: { docId => { versions: [ {version, content, status} ] } }
    this.store = new Map();
    // fila de relatórios de bug gerados pela validação
    this.bugQueue = [];
  }

  // Salva uma versão de documento rapidamente (fast path) e agenda validação em background.
  // options.strictValidation: se true, quando a validação falhar, revertemos (status 'reverted').
  // Se false, marcamos como 'quarantined' e não bloqueamos a experiência do usuário.
  saveVersion(userId, docId, content, options = { strictValidation: false }) {
    const doc = this.store.get(docId) || { versions: [] };
    const versionNumber = doc.versions.length + 1;
    const entry = { version: versionNumber, content, status: 'active', author: userId, createdAt: Date.now() };
    doc.versions.push(entry);
    this.store.set(docId, doc);

    // Agendamos validação assíncrona (background)
    // Usamos setTimeout(..., 0) para simular trabalho assíncrono sem bloquear.
    setTimeout(async () => {
      try {
        const result = await this.validator(content);
        if (!result || result.valid === undefined) {
          // Se validator for mal formado, tratamos como falha segura
          result = { valid: false, errors: ['validator_error'] };
        }
        if (!result.valid) {
          if (options.strictValidation) {
            entry.status = 'reverted';
          } else {
            entry.status = 'quarantined';
          }
          this._enqueueBugReport(docId, versionNumber, result.errors || []);
        } else {
          // Em caso de sucesso, mantemos 'active' (poderíamos também promover ou marcar como 'validated')
          entry.status = 'active';
        }
      } catch (err) {
        // Falha inesperada na validação: marcar em quarentena e gerar bug
        entry.status = 'quarantined';
        this._enqueueBugReport(docId, versionNumber, [String(err)]);
      }
    }, 0);

    // Retornamos informação imediata ao usuário — experiência rápida.
    return { version: versionNumber, status: entry.status };
  }

  getVersion(docId, versionNumber) {
    const doc = this.store.get(docId);
    if (!doc) return null;
    return doc.versions.find(v => v.version === versionNumber) || null;
  }

  listVersions(docId) {
    const doc = this.store.get(docId);
    return doc ? [...doc.versions] : [];
  }

  _enqueueBugReport(docId, version, errors) {
    const id = `bug-${Date.now()}-${Math.floor(Math.random() * 100000)}`;
    const report = { id, docId, version, errors, createdAt: Date.now() };
    this.bugQueue.push(report);
  }

  getBugReports() {
    return [...this.bugQueue];
  }

  clear() {
    this.store.clear();
    this.bugQueue = [];
  }
}

module.exports = { CollabService };

const { CollabService } = require('../src/collab');

jest.useFakeTimers();

describe('CollabService - velocidade vs qualidade', () => {
  afterEach(() => {
    jest.clearAllTimers();
    jest.resetAllMocks();
  });

  test('salva imediatamente (fast path) e valida em background (validação passa)', async () => {
    const validator = jest.fn(async (content) => ({ valid: true }));
    const svc = new CollabService(validator);

    const res = svc.saveVersion('user1', 'docA', 'conteúdo OK', { strictValidation: false });

    // Retorno imediato deve apontar para versão ativa
    expect(res).toEqual({ version: 1, status: 'active' });

    // Ainda não executamos validação até avançar timers
    expect(validator).not.toHaveBeenCalled();

    // Executa timer pendente (validação)
    jest.runOnlyPendingTimers();
    // Allow pending promises to resolve
    await Promise.resolve();

    // Validador foi chamado e status continua 'active'
    expect(validator).toHaveBeenCalledTimes(1);
    const v = svc.getVersion('docA', 1);
    expect(v.status).toBe('active');
    expect(svc.getBugReports().length).toBe(0);
  });

  test('quando validação falha e strictValidation=false -> marca quarantined e gera bug', async () => {
    const validator = jest.fn(async (content) => ({ valid: false, errors: ['syntax_error'] }));
    const svc = new CollabService(validator);

    const res = svc.saveVersion('user2', 'docB', 'conteúdo com bug', { strictValidation: false });
    expect(res.status).toBe('active');

    jest.runOnlyPendingTimers();
    await Promise.resolve();

    const v = svc.getVersion('docB', 1);
    expect(v.status).toBe('quarantined');
    const bugs = svc.getBugReports();
    expect(bugs.length).toBe(1);
    expect(bugs[0].docId).toBe('docB');
    expect(bugs[0].errors).toContain('syntax_error');
  });

  test('quando validação falha e strictValidation=true -> reverte imediatamente na validação (rollback) e gera bug', async () => {
    const validator = jest.fn(async (content) => ({ valid: false, errors: ['critical_rule'] }));
    const svc = new CollabService(validator);

    const res = svc.saveVersion('user3', 'docC', 'conteúdo crítico', { strictValidation: true });
    expect(res.status).toBe('active');

    jest.runOnlyPendingTimers();
    await Promise.resolve();

    const v = svc.getVersion('docC', 1);
    expect(v.status).toBe('reverted');
    const bugs = svc.getBugReports();
    expect(bugs.length).toBe(1);
    expect(bugs[0].errors).toContain('critical_rule');
  });

  test('múltiplas versões e manutenção do histórico', async () => {
    const validator = jest.fn(async (content) => ({ valid: content !== 'bad' }));
    const svc = new CollabService(validator);

    svc.saveVersion('u', 'docD', 'first', {});
    svc.saveVersion('u', 'docD', 'bad', { strictValidation: false });
    svc.saveVersion('u', 'docD', 'third', { strictValidation: true });

    jest.runOnlyPendingTimers();
    await Promise.resolve();

    const versions = svc.listVersions('docD').map(v => ({ version: v.version, status: v.status }));
    // first -> active, second -> quarantined, third -> reverted (strict + bad)
    expect(versions).toEqual(
      expect.arrayContaining([
        { version: 1, status: 'active' },
        { version: 2, status: 'quarantined' },
        { version: 3, status: 'reverted' },
      ])
    );
    expect(svc.getBugReports().length).toBe(2); // second and third geraram bugs
  });
});

{
  "name": "collab-speed-quality-example",
  "version": "1.0.0",
  "description": "Exemplo mostrando estratégia fast-path + validação assíncrona para equilibrar velocidade e qualidade",
  "main": "src/collab.js",
  "scripts": {
    "test": "jest"
  },
  "devDependencies": {
    "jest": "^29.0.0"
  }
}

# Exemplo: Equilibrando Velocidade e Qualidade (Fast Path + Validação Assíncrona)

Este repositório demonstra uma estratégia para conciliar velocidade (resposta imediata ao usuário) e qualidade (validações e quarentena/rollback) em uma ferramenta de colaboração.

O que foi implementado
- src/collab.js: serviço que salva versões rapidamente e executa validação em background.
  - Fast path: retorno imediato ao usuário.
  - Validação assíncrona: marca versões como `active`, `quarantined` ou `reverted`.
  - Geração de relatórios de bug em uma fila interna (`bugQueue`).
- __tests__/collab.test.js: testes automatizados com Jest cobrindo cenários principais.
- package.json: script `npm test` para rodar os testes.

Como rodar
1. npm install
2. npm test

Estrutura de arquivos
- src/collab.js — implementação do serviço
- __tests__/collab.test.js — testes unitários
- package.json — dependências e scripts
- README.md — esta explicação

Próximos passos sugeridos
- Persistir em banco em vez de memória.
- Integração com filas e workers para validação distribuída.
- Criar automação para transformar `bugQueue` em issues/tickets.
- Coletar métricas e dashboards (latência, taxa de quarentena, tempo até validação).

Resumo curto
Salve o código em `src/collab.js`, os testes em `__tests__/collab.test.js`, e as explicações em `README.md` na raiz do repositório.