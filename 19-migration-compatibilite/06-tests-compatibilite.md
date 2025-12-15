🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.6 Tests de compatibilité

> **Niveau** : Avancé / Expert  
> **Durée estimée** : 3-4 heures  
> **Prérequis** : Expérience en tests logiciels, connaissance des environnements de staging, maîtrise des outils de monitoring MariaDB

## 🎯 Objectifs d'apprentissage

À l'issue de cette section, vous serez capable de :

- Concevoir une stratégie de tests complète pour valider une migration MariaDB
- Mettre en place des environnements de test représentatifs de la production
- Automatiser les tests de régression fonctionnels et de performance
- Comparer objectivement les performances avant/après migration
- Valider la compatibilité en conditions réelles avec un risque maîtrisé
- Documenter et tracer les résultats de tests pour l'audit et le go/no-go

---

## Introduction

"Ça marche en dev" est sans doute la phrase la plus dangereuse en informatique. Dans le contexte d'une migration de base de données, elle peut se traduire par des heures de debug en production, des rollbacks d'urgence, et une perte de confiance des équipes métier.

Les tests de compatibilité constituent le filet de sécurité indispensable de toute migration. Ils permettent de détecter les problèmes avant qu'ils n'impactent les utilisateurs, de quantifier les risques, et de prendre des décisions éclairées sur le go/no-go de la migration.

Cette section présente une méthodologie structurée de tests, des outils pratiques, et des critères objectifs pour valider la compatibilité de votre migration vers MariaDB.

---

## Stratégie de tests : Vue d'ensemble

### Pyramide des tests de migration

```
Pyramide des tests de migration
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

                    ╱╲
                   ╱  ╲
                  ╱    ╲
                 ╱ PROD ╲         ← Tests en production (canary, shadow)
                ╱ VALID. ╲           Risque: Réel | Couverture: Maximale
               ╱──────────╲
              ╱            ╲
             ╱   CHARGE     ╲     ← Tests de charge et performance
            ╱    PERF        ╲       Risque: Aucun | Couverture: Performance
           ╱──────────────────╲
          ╱                    ╲
         ╱   INTÉGRATION        ╲  ← Tests d'intégration E2E
        ╱    END-TO-END          ╲    Risque: Aucun | Couverture: Fonctionnelle
       ╱──────────────────────────╲
      ╱                            ╲
     ╱      FONCTIONNELS            ╲ ← Tests fonctionnels unitaires
    ╱       UNITAIRES                ╲   Risque: Aucun | Couverture: Requêtes
   ╱──────────────────────────────────╲
  ╱                                    ╲
 ╱         COMPATIBILITÉ                ╲ ← Tests de compatibilité technique
╱          TECHNIQUE                     ╲   Risque: Aucun | Couverture: Infra
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

        PLUS DE TESTS ─────────────────▶
        MOINS DE RISQUE ◀───────────────
```

### Phases de tests

| Phase | Objectif | Environnement | Durée type |
|-------|----------|---------------|------------|
| **1. Technique** | Compatibilité infra, connecteurs | Dev/Sandbox | 1-2 jours |
| **2. Fonctionnel** | Requêtes SQL, procédures | Staging | 3-5 jours |
| **3. Intégration** | Application E2E | Staging | 5-10 jours |
| **4. Performance** | Benchmarks, charge | Pré-prod | 3-5 jours |
| **5. Validation** | Canary, shadow traffic | Production | 1-7 jours |

---

## Phase 1 : Tests de compatibilité technique

### Checklist de compatibilité technique

```bash
#!/bin/bash
# technical_compatibility_check.sh
# Vérification de compatibilité technique MariaDB 11.8

set -e

MARIADB_HOST="${1:-localhost}"
MARIADB_USER="${2:-root}"
MARIADB_PASS="${3:-}"

echo "═══════════════════════════════════════════════════════"
echo "   TESTS DE COMPATIBILITÉ TECHNIQUE - MariaDB 11.8"
echo "═══════════════════════════════════════════════════════"
echo ""

# Fonction de test
run_test() {
    local test_name="$1"
    local test_cmd="$2"
    
    echo -n "[$test_name] "
    if eval "$test_cmd" > /dev/null 2>&1; then
        echo "✅ PASS"
        return 0
    else
        echo "❌ FAIL"
        return 1
    fi
}

# Compteurs
PASSED=0
FAILED=0

# 1. Test de connexion
echo "── Connectivité ──────────────────────────────────────"
if run_test "Connexion basique" \
    "mariadb -h $MARIADB_HOST -u $MARIADB_USER -p$MARIADB_PASS -e 'SELECT 1'"; then
    PASSED=$((PASSED+1))
else
    FAILED=$((FAILED+1))
fi

# 2. Version
echo ""
echo "── Version et Configuration ─────────────────────────"
VERSION=$(mariadb -h $MARIADB_HOST -u $MARIADB_USER -p$MARIADB_PASS -N -e "SELECT VERSION()")
echo "   Version détectée: $VERSION"

if [[ "$VERSION" == *"11.8"* ]]; then
    echo "   ✅ MariaDB 11.8 confirmé"
    PASSED=$((PASSED+1))
else
    echo "   ⚠️ Version différente de 11.8"
fi

# 3. Charset par défaut
echo ""
echo "── Charset et Collation ─────────────────────────────"
CHARSET=$(mariadb -h $MARIADB_HOST -u $MARIADB_USER -p$MARIADB_PASS -N -e \
    "SELECT @@character_set_server")
COLLATION=$(mariadb -h $MARIADB_HOST -u $MARIADB_USER -p$MARIADB_PASS -N -e \
    "SELECT @@collation_server")

echo "   Charset serveur: $CHARSET"
echo "   Collation serveur: $COLLATION"

if [[ "$CHARSET" == "utf8mb4" ]]; then
    echo "   ✅ utf8mb4 par défaut (11.8)"
    PASSED=$((PASSED+1))
else
    echo "   ⚠️ Charset non standard"
fi

# 4. TLS
echo ""
echo "── Sécurité TLS ─────────────────────────────────────"
SSL_STATUS=$(mariadb -h $MARIADB_HOST -u $MARIADB_USER -p$MARIADB_PASS -N -e \
    "SHOW VARIABLES LIKE 'have_ssl'" | awk '{print $2}')

if [[ "$SSL_STATUS" == "YES" ]]; then
    echo "   ✅ TLS disponible"
    PASSED=$((PASSED+1))
else
    echo "   ⚠️ TLS non disponible"
    FAILED=$((FAILED+1))
fi

# 5. Moteurs de stockage
echo ""
echo "── Moteurs de Stockage ──────────────────────────────"
ENGINES=$(mariadb -h $MARIADB_HOST -u $MARIADB_USER -p$MARIADB_PASS -N -e \
    "SELECT ENGINE FROM information_schema.ENGINES WHERE SUPPORT IN ('YES','DEFAULT')" | tr '\n' ', ')
echo "   Moteurs disponibles: $ENGINES"

if run_test "InnoDB disponible" \
    "mariadb -h $MARIADB_HOST -u $MARIADB_USER -p$MARIADB_PASS -e \"SELECT 1 FROM information_schema.ENGINES WHERE ENGINE='InnoDB' AND SUPPORT='DEFAULT'\" | grep -q 1"; then
    PASSED=$((PASSED+1))
else
    FAILED=$((FAILED+1))
fi

# 6. Fonctionnalités SQL
echo ""
echo "── Fonctionnalités SQL ──────────────────────────────"

# JSON
if run_test "JSON support" \
    "mariadb -h $MARIADB_HOST -u $MARIADB_USER -p$MARIADB_PASS -e \"SELECT JSON_OBJECT('test', 1)\""; then
    PASSED=$((PASSED+1))
else
    FAILED=$((FAILED+1))
fi

# Window Functions
if run_test "Window Functions" \
    "mariadb -h $MARIADB_HOST -u $MARIADB_USER -p$MARIADB_PASS -e \"SELECT ROW_NUMBER() OVER () FROM (SELECT 1) t\""; then
    PASSED=$((PASSED+1))
else
    FAILED=$((FAILED+1))
fi

# CTE Recursive
if run_test "CTE Récursives" \
    "mariadb -h $MARIADB_HOST -u $MARIADB_USER -p$MARIADB_PASS -e \"WITH RECURSIVE cte AS (SELECT 1 AS n UNION ALL SELECT n+1 FROM cte WHERE n<3) SELECT * FROM cte\""; then
    PASSED=$((PASSED+1))
else
    FAILED=$((FAILED+1))
fi

# System Versioning
if run_test "System Versioning" \
    "mariadb -h $MARIADB_HOST -u $MARIADB_USER -p$MARIADB_PASS -e \"CREATE TEMPORARY TABLE test_sv (id INT) WITH SYSTEM VERSIONING; DROP TABLE test_sv\""; then
    PASSED=$((PASSED+1))
else
    FAILED=$((FAILED+1))
fi

# 7. Timestamp étendu (11.8)
echo ""
echo "── Nouveautés 11.8 ──────────────────────────────────"

if run_test "TIMESTAMP > 2038" \
    "mariadb -h $MARIADB_HOST -u $MARIADB_USER -p$MARIADB_PASS -e \"SELECT TIMESTAMP'2050-01-01 00:00:00'\""; then
    PASSED=$((PASSED+1))
    echo "   ✅ Extension TIMESTAMP 2106 supportée"
else
    FAILED=$((FAILED+1))
fi

# Résumé
echo ""
echo "═══════════════════════════════════════════════════════"
echo "   RÉSUMÉ"
echo "═══════════════════════════════════════════════════════"
echo "   Tests réussis: $PASSED"
echo "   Tests échoués: $FAILED"
echo ""

if [ $FAILED -eq 0 ]; then
    echo "   ✅ Tous les tests techniques ont réussi"
    exit 0
else
    echo "   ⚠️ Certains tests ont échoué - Investigation requise"
    exit 1
fi
```

### Test de connectivité depuis l'application

```python
# connectivity_test.py
# Test de connectivité multi-langage simulé

import subprocess
import json
from datetime import datetime

class ConnectivityTester:
    """Testeur de connectivité pour différents types de connexions"""
    
    def __init__(self, host, port, user, password, database):
        self.config = {
            'host': host,
            'port': port,
            'user': user,
            'password': password,
            'database': database
        }
        self.results = []
    
    def test_python_mariadb(self):
        """Test connecteur Python mariadb"""
        try:
            import mariadb
            conn = mariadb.connect(**self.config)
            cursor = conn.cursor()
            cursor.execute("SELECT VERSION(), CONNECTION_ID()")
            version, conn_id = cursor.fetchone()
            conn.close()
            return {'status': 'PASS', 'version': version, 'connection_id': conn_id}
        except Exception as e:
            return {'status': 'FAIL', 'error': str(e)}
    
    def test_python_pymysql(self):
        """Test connecteur PyMySQL"""
        try:
            import pymysql
            conn = pymysql.connect(
                host=self.config['host'],
                port=self.config['port'],
                user=self.config['user'],
                password=self.config['password'],
                database=self.config['database']
            )
            cursor = conn.cursor()
            cursor.execute("SELECT VERSION()")
            version = cursor.fetchone()[0]
            conn.close()
            return {'status': 'PASS', 'version': version}
        except Exception as e:
            return {'status': 'FAIL', 'error': str(e)}
    
    def test_sqlalchemy(self):
        """Test SQLAlchemy"""
        try:
            from sqlalchemy import create_engine, text
            url = f"mariadb+mariadbconnector://{self.config['user']}:{self.config['password']}@{self.config['host']}:{self.config['port']}/{self.config['database']}"
            engine = create_engine(url)
            with engine.connect() as conn:
                result = conn.execute(text("SELECT VERSION()"))
                version = result.fetchone()[0]
            return {'status': 'PASS', 'version': version}
        except Exception as e:
            return {'status': 'FAIL', 'error': str(e)}
    
    def run_all_tests(self):
        """Exécute tous les tests de connectivité"""
        tests = [
            ('Python mariadb', self.test_python_mariadb),
            ('Python PyMySQL', self.test_python_pymysql),
            ('SQLAlchemy', self.test_sqlalchemy),
        ]
        
        print("=" * 60)
        print("TESTS DE CONNECTIVITÉ")
        print("=" * 60)
        print(f"Host: {self.config['host']}:{self.config['port']}")
        print(f"Database: {self.config['database']}")
        print("-" * 60)
        
        for name, test_func in tests:
            print(f"\n[{name}]")
            result = test_func()
            self.results.append({'test': name, **result})
            
            if result['status'] == 'PASS':
                print(f"  ✅ PASS - Version: {result.get('version', 'N/A')}")
            else:
                print(f"  ❌ FAIL - Error: {result.get('error', 'Unknown')}")
        
        # Résumé
        passed = sum(1 for r in self.results if r['status'] == 'PASS')
        failed = len(self.results) - passed
        
        print("\n" + "=" * 60)
        print(f"RÉSUMÉ: {passed} PASS, {failed} FAIL")
        print("=" * 60)
        
        return self.results


if __name__ == '__main__':
    tester = ConnectivityTester(
        host='localhost',
        port=3306,
        user='test_user',
        password='test_password',
        database='test_db'
    )
    tester.run_all_tests()
```

---

## Phase 2 : Tests fonctionnels

### Framework de tests SQL

```python
# sql_functional_tests.py
# Framework de tests fonctionnels SQL pour migration MariaDB

import mariadb
import hashlib
import json
from dataclasses import dataclass
from typing import List, Dict, Any, Optional
from datetime import datetime

@dataclass
class TestCase:
    """Définition d'un cas de test SQL"""
    id: str
    name: str
    query: str
    expected_columns: Optional[List[str]] = None
    expected_row_count: Optional[int] = None
    expected_checksum: Optional[str] = None
    max_execution_time_ms: Optional[int] = None
    tags: List[str] = None

@dataclass
class TestResult:
    """Résultat d'un cas de test"""
    test_id: str
    status: str  # PASS, FAIL, ERROR
    execution_time_ms: float
    row_count: int
    checksum: str
    columns: List[str]
    error_message: Optional[str] = None
    details: Dict[str, Any] = None


class SQLFunctionalTestRunner:
    """Exécuteur de tests fonctionnels SQL"""
    
    def __init__(self, connection_config: Dict):
        self.config = connection_config
        self.conn = None
        self.results: List[TestResult] = []
    
    def connect(self):
        """Établit la connexion à MariaDB"""
        self.conn = mariadb.connect(**self.config)
    
    def disconnect(self):
        """Ferme la connexion"""
        if self.conn:
            self.conn.close()
    
    def _compute_checksum(self, rows: List[tuple]) -> str:
        """Calcule un checksum des résultats"""
        data = json.dumps(rows, default=str, sort_keys=True)
        return hashlib.md5(data.encode()).hexdigest()
    
    def run_test(self, test_case: TestCase) -> TestResult:
        """Exécute un cas de test"""
        cursor = self.conn.cursor()
        
        start_time = datetime.now()
        error_message = None
        status = "PASS"
        rows = []
        columns = []
        
        try:
            cursor.execute(test_case.query)
            
            # Récupérer les métadonnées
            if cursor.description:
                columns = [col[0] for col in cursor.description]
                rows = cursor.fetchall()
            
        except Exception as e:
            status = "ERROR"
            error_message = str(e)
        
        end_time = datetime.now()
        execution_time_ms = (end_time - start_time).total_seconds() * 1000
        
        row_count = len(rows)
        checksum = self._compute_checksum(rows) if rows else ""
        
        # Validations
        details = {}
        
        if status != "ERROR":
            # Vérifier le nombre de colonnes
            if test_case.expected_columns:
                if columns != test_case.expected_columns:
                    status = "FAIL"
                    details['columns_mismatch'] = {
                        'expected': test_case.expected_columns,
                        'actual': columns
                    }
            
            # Vérifier le nombre de lignes
            if test_case.expected_row_count is not None:
                if row_count != test_case.expected_row_count:
                    status = "FAIL"
                    details['row_count_mismatch'] = {
                        'expected': test_case.expected_row_count,
                        'actual': row_count
                    }
            
            # Vérifier le checksum
            if test_case.expected_checksum:
                if checksum != test_case.expected_checksum:
                    status = "FAIL"
                    details['checksum_mismatch'] = {
                        'expected': test_case.expected_checksum,
                        'actual': checksum
                    }
            
            # Vérifier le temps d'exécution
            if test_case.max_execution_time_ms:
                if execution_time_ms > test_case.max_execution_time_ms:
                    status = "FAIL"
                    details['timeout'] = {
                        'max_allowed_ms': test_case.max_execution_time_ms,
                        'actual_ms': execution_time_ms
                    }
        
        cursor.close()
        
        result = TestResult(
            test_id=test_case.id,
            status=status,
            execution_time_ms=execution_time_ms,
            row_count=row_count,
            checksum=checksum,
            columns=columns,
            error_message=error_message,
            details=details if details else None
        )
        
        self.results.append(result)
        return result
    
    def run_tests(self, test_cases: List[TestCase]) -> List[TestResult]:
        """Exécute une liste de cas de test"""
        self.connect()
        
        print("=" * 70)
        print("TESTS FONCTIONNELS SQL")
        print("=" * 70)
        
        for test_case in test_cases:
            print(f"\n[{test_case.id}] {test_case.name}")
            result = self.run_test(test_case)
            
            if result.status == "PASS":
                print(f"  ✅ PASS ({result.execution_time_ms:.2f}ms, {result.row_count} rows)")
            elif result.status == "FAIL":
                print(f"  ❌ FAIL ({result.execution_time_ms:.2f}ms)")
                if result.details:
                    for key, value in result.details.items():
                        print(f"     - {key}: {value}")
            else:
                print(f"  💥 ERROR: {result.error_message}")
        
        self.disconnect()
        
        # Résumé
        passed = sum(1 for r in self.results if r.status == "PASS")
        failed = sum(1 for r in self.results if r.status == "FAIL")
        errors = sum(1 for r in self.results if r.status == "ERROR")
        
        print("\n" + "=" * 70)
        print(f"RÉSUMÉ: {passed} PASS, {failed} FAIL, {errors} ERROR")
        print("=" * 70)
        
        return self.results
    
    def export_results(self, filepath: str):
        """Exporte les résultats en JSON"""
        data = {
            'timestamp': datetime.now().isoformat(),
            'total_tests': len(self.results),
            'passed': sum(1 for r in self.results if r.status == "PASS"),
            'failed': sum(1 for r in self.results if r.status == "FAIL"),
            'errors': sum(1 for r in self.results if r.status == "ERROR"),
            'results': [
                {
                    'test_id': r.test_id,
                    'status': r.status,
                    'execution_time_ms': r.execution_time_ms,
                    'row_count': r.row_count,
                    'checksum': r.checksum,
                    'error': r.error_message,
                    'details': r.details
                }
                for r in self.results
            ]
        }
        
        with open(filepath, 'w') as f:
            json.dump(data, f, indent=2)
        
        print(f"\nRésultats exportés vers: {filepath}")


# Exemple d'utilisation
if __name__ == '__main__':
    # Définition des cas de test
    test_cases = [
        TestCase(
            id="TC001",
            name="Sélection basique",
            query="SELECT 1 AS value",
            expected_columns=['value'],
            expected_row_count=1
        ),
        TestCase(
            id="TC002",
            name="Fonctions JSON",
            query="SELECT JSON_EXTRACT('{\"name\": \"test\"}', '$.name') AS result",
            expected_columns=['result'],
            expected_row_count=1
        ),
        TestCase(
            id="TC003",
            name="Window Functions",
            query="""
                SELECT 
                    ROW_NUMBER() OVER (ORDER BY n) AS rn,
                    n
                FROM (SELECT 1 AS n UNION SELECT 2 UNION SELECT 3) t
            """,
            expected_row_count=3
        ),
        TestCase(
            id="TC004",
            name="CTE Récursive",
            query="""
                WITH RECURSIVE numbers AS (
                    SELECT 1 AS n
                    UNION ALL
                    SELECT n + 1 FROM numbers WHERE n < 5
                )
                SELECT SUM(n) AS total FROM numbers
            """,
            expected_row_count=1
        ),
        TestCase(
            id="TC005",
            name="Timestamp étendu 2050",
            query="SELECT TIMESTAMP'2050-12-31 23:59:59' AS future_date",
            expected_row_count=1
        ),
    ]
    
    # Configuration
    config = {
        'host': 'localhost',
        'port': 3306,
        'user': 'test_user',
        'password': 'test_password',
        'database': 'test_db'
    }
    
    # Exécution
    runner = SQLFunctionalTestRunner(config)
    results = runner.run_tests(test_cases)
    runner.export_results('functional_test_results.json')
```

### Comparaison source/cible

```python
# compare_query_results.py
# Comparaison des résultats de requêtes entre source et cible

import mariadb
import hashlib
import json
from typing import Dict, List, Tuple, Any
from dataclasses import dataclass

@dataclass
class ComparisonResult:
    """Résultat de comparaison entre source et cible"""
    query_id: str
    query: str
    source_row_count: int
    target_row_count: int
    source_checksum: str
    target_checksum: str
    match: bool
    source_time_ms: float
    target_time_ms: float
    performance_delta_pct: float


class QueryComparator:
    """Compare les résultats de requêtes entre deux bases"""
    
    def __init__(self, source_config: Dict, target_config: Dict):
        self.source_config = source_config
        self.target_config = target_config
        self.results: List[ComparisonResult] = []
    
    def _execute_query(self, config: Dict, query: str) -> Tuple[List, float, str]:
        """Exécute une requête et retourne résultats, temps, checksum"""
        import time
        
        conn = mariadb.connect(**config)
        cursor = conn.cursor()
        
        start = time.time()
        cursor.execute(query)
        rows = cursor.fetchall()
        end = time.time()
        
        cursor.close()
        conn.close()
        
        execution_time_ms = (end - start) * 1000
        checksum = hashlib.md5(
            json.dumps(rows, default=str, sort_keys=True).encode()
        ).hexdigest()
        
        return rows, execution_time_ms, checksum
    
    def compare_query(self, query_id: str, query: str) -> ComparisonResult:
        """Compare une requête entre source et cible"""
        
        # Exécuter sur la source
        source_rows, source_time, source_checksum = self._execute_query(
            self.source_config, query
        )
        
        # Exécuter sur la cible
        target_rows, target_time, target_checksum = self._execute_query(
            self.target_config, query
        )
        
        # Calculer le delta de performance
        if source_time > 0:
            perf_delta = ((target_time - source_time) / source_time) * 100
        else:
            perf_delta = 0
        
        result = ComparisonResult(
            query_id=query_id,
            query=query[:100] + "..." if len(query) > 100 else query,
            source_row_count=len(source_rows),
            target_row_count=len(target_rows),
            source_checksum=source_checksum,
            target_checksum=target_checksum,
            match=(source_checksum == target_checksum),
            source_time_ms=source_time,
            target_time_ms=target_time,
            performance_delta_pct=perf_delta
        )
        
        self.results.append(result)
        return result
    
    def compare_queries(self, queries: Dict[str, str]) -> List[ComparisonResult]:
        """Compare une liste de requêtes"""
        
        print("=" * 80)
        print("COMPARAISON SOURCE / CIBLE")
        print("=" * 80)
        print(f"Source: {self.source_config['host']}")
        print(f"Cible:  {self.target_config['host']}")
        print("-" * 80)
        
        for query_id, query in queries.items():
            print(f"\n[{query_id}]")
            result = self.compare_query(query_id, query)
            
            # Affichage
            status = "✅ MATCH" if result.match else "❌ MISMATCH"
            print(f"  {status}")
            print(f"  Rows: {result.source_row_count} → {result.target_row_count}")
            print(f"  Time: {result.source_time_ms:.2f}ms → {result.target_time_ms:.2f}ms ({result.performance_delta_pct:+.1f}%)")
            
            if not result.match:
                print(f"  Source checksum: {result.source_checksum}")
                print(f"  Target checksum: {result.target_checksum}")
        
        # Résumé
        matched = sum(1 for r in self.results if r.match)
        mismatched = len(self.results) - matched
        avg_perf_delta = sum(r.performance_delta_pct for r in self.results) / len(self.results)
        
        print("\n" + "=" * 80)
        print("RÉSUMÉ")
        print("=" * 80)
        print(f"  Requêtes correspondantes: {matched}/{len(self.results)}")
        print(f"  Requêtes divergentes: {mismatched}")
        print(f"  Delta performance moyen: {avg_perf_delta:+.1f}%")
        
        return self.results
    
    def generate_report(self, filepath: str):
        """Génère un rapport détaillé"""
        report = {
            'timestamp': __import__('datetime').datetime.now().isoformat(),
            'source': self.source_config['host'],
            'target': self.target_config['host'],
            'summary': {
                'total_queries': len(self.results),
                'matched': sum(1 for r in self.results if r.match),
                'mismatched': sum(1 for r in self.results if not r.match),
                'avg_performance_delta_pct': sum(r.performance_delta_pct for r in self.results) / len(self.results) if self.results else 0
            },
            'results': [
                {
                    'query_id': r.query_id,
                    'query': r.query,
                    'match': r.match,
                    'source_rows': r.source_row_count,
                    'target_rows': r.target_row_count,
                    'source_time_ms': r.source_time_ms,
                    'target_time_ms': r.target_time_ms,
                    'performance_delta_pct': r.performance_delta_pct
                }
                for r in self.results
            ]
        }
        
        with open(filepath, 'w') as f:
            json.dump(report, f, indent=2)
        
        print(f"\nRapport généré: {filepath}")


# Exemple d'utilisation
if __name__ == '__main__':
    source_config = {
        'host': 'mysql-source',
        'port': 3306,
        'user': 'test_user',
        'password': 'test_password',
        'database': 'mydb'
    }
    
    target_config = {
        'host': 'mariadb-target',
        'port': 3306,
        'user': 'test_user',
        'password': 'test_password',
        'database': 'mydb'
    }
    
    queries = {
        'Q001': "SELECT COUNT(*) FROM users",
        'Q002': "SELECT COUNT(*) FROM orders WHERE status = 'completed'",
        'Q003': "SELECT customer_id, SUM(total) FROM orders GROUP BY customer_id ORDER BY customer_id LIMIT 100",
        # Ajouter vos requêtes critiques ici
    }
    
    comparator = QueryComparator(source_config, target_config)
    comparator.compare_queries(queries)
    comparator.generate_report('comparison_report.json')
```

---

## Phase 3 : Tests d'intégration E2E

### Architecture de test d'intégration

```
Architecture des tests E2E
━━━━━━━━━━━━━━━━━━━━━━━━━━

┌─────────────────────────────────────────────────────────────┐
│                    ORCHESTRATEUR DE TESTS                   │
│                    (pytest / Jest / JUnit)                  │
└─────────────────────────────────────────────────────────────┘
                              │
           ┌──────────────────┼──────────────────┐
           │                  │                  │
           ▼                  ▼                  ▼
    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
    │  API Tests  │    │  UI Tests   │    │ Batch Tests │
    │             │    │ (Selenium)  │    │             │
    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
           │                  │                  │
           └──────────────────┼──────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │   APPLICATION   │
                    │   (Staging)     │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  MariaDB 11.8   │
                    │   (Staging)     │
                    └─────────────────┘
```

### Configuration Docker Compose pour tests

```yaml
# docker-compose.test.yml
# Environnement de test d'intégration

version: '3.8'

services:
  # MariaDB 11.8 pour tests
  mariadb-test:
    image: mariadb:11.8
    container_name: mariadb-test
    environment:
      MYSQL_ROOT_PASSWORD: test_root_password
      MYSQL_DATABASE: test_db
      MYSQL_USER: test_user
      MYSQL_PASSWORD: test_password
    ports:
      - "3307:3306"
    volumes:
      - ./init-scripts:/docker-entrypoint-initdb.d:ro
      - mariadb-test-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mariadb-admin", "ping", "-h", "localhost", "-u", "root", "-ptest_root_password"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - test-network

  # Application à tester
  app-test:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: app-test
    environment:
      DATABASE_HOST: mariadb-test
      DATABASE_PORT: 3306
      DATABASE_NAME: test_db
      DATABASE_USER: test_user
      DATABASE_PASSWORD: test_password
    depends_on:
      mariadb-test:
        condition: service_healthy
    networks:
      - test-network

  # Runner de tests
  test-runner:
    build:
      context: ./tests
      dockerfile: Dockerfile.test
    container_name: test-runner
    environment:
      APP_URL: http://app-test:8080
      DATABASE_HOST: mariadb-test
      DATABASE_PORT: 3306
      DATABASE_NAME: test_db
      DATABASE_USER: test_user
      DATABASE_PASSWORD: test_password
    depends_on:
      - app-test
    volumes:
      - ./tests:/tests
      - ./test-results:/results
    command: pytest /tests -v --junitxml=/results/junit.xml
    networks:
      - test-network

networks:
  test-network:
    driver: bridge

volumes:
  mariadb-test-data:
```

### Script de tests E2E avec pytest

```python
# tests/test_e2e_integration.py
# Tests d'intégration E2E pour validation migration

import pytest
import requests
import mariadb
import time
from datetime import datetime

# Configuration
APP_URL = "http://app-test:8080"
DB_CONFIG = {
    'host': 'mariadb-test',
    'port': 3306,
    'user': 'test_user',
    'password': 'test_password',
    'database': 'test_db'
}


@pytest.fixture(scope="module")
def db_connection():
    """Fixture de connexion à la base de données"""
    conn = mariadb.connect(**DB_CONFIG)
    yield conn
    conn.close()


@pytest.fixture(scope="module")
def api_session():
    """Fixture de session HTTP"""
    session = requests.Session()
    yield session
    session.close()


class TestDatabaseConnectivity:
    """Tests de connectivité base de données"""
    
    def test_connection_established(self, db_connection):
        """Vérifie que la connexion est établie"""
        cursor = db_connection.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        assert result[0] == 1
    
    def test_mariadb_version(self, db_connection):
        """Vérifie la version MariaDB"""
        cursor = db_connection.cursor()
        cursor.execute("SELECT VERSION()")
        version = cursor.fetchone()[0]
        assert "11.8" in version, f"Expected MariaDB 11.8, got {version}"
    
    def test_utf8mb4_charset(self, db_connection):
        """Vérifie le charset par défaut"""
        cursor = db_connection.cursor()
        cursor.execute("SELECT @@character_set_server")
        charset = cursor.fetchone()[0]
        assert charset == "utf8mb4"


class TestApplicationAPI:
    """Tests de l'API applicative"""
    
    def test_health_endpoint(self, api_session):
        """Vérifie l'endpoint de santé"""
        response = api_session.get(f"{APP_URL}/health")
        assert response.status_code == 200
        data = response.json()
        assert data.get('database') == 'connected'
    
    def test_create_user(self, api_session, db_connection):
        """Test création d'utilisateur via API"""
        # Créer via API
        user_data = {
            'email': f'test_{int(time.time())}@example.com',
            'name': 'Test User'
        }
        response = api_session.post(f"{APP_URL}/api/users", json=user_data)
        assert response.status_code == 201
        created_user = response.json()
        
        # Vérifier en base
        cursor = db_connection.cursor()
        cursor.execute(
            "SELECT email, name FROM users WHERE id = %s",
            (created_user['id'],)
        )
        row = cursor.fetchone()
        assert row is not None
        assert row[0] == user_data['email']
        assert row[1] == user_data['name']
    
    def test_read_users(self, api_session):
        """Test lecture des utilisateurs via API"""
        response = api_session.get(f"{APP_URL}/api/users")
        assert response.status_code == 200
        users = response.json()
        assert isinstance(users, list)
    
    def test_update_user(self, api_session, db_connection):
        """Test mise à jour d'utilisateur"""
        # Créer un utilisateur
        user_data = {'email': f'update_test_{int(time.time())}@example.com', 'name': 'Original'}
        response = api_session.post(f"{APP_URL}/api/users", json=user_data)
        user_id = response.json()['id']
        
        # Mettre à jour
        update_data = {'name': 'Updated Name'}
        response = api_session.put(f"{APP_URL}/api/users/{user_id}", json=update_data)
        assert response.status_code == 200
        
        # Vérifier
        cursor = db_connection.cursor()
        cursor.execute("SELECT name FROM users WHERE id = %s", (user_id,))
        assert cursor.fetchone()[0] == 'Updated Name'
    
    def test_delete_user(self, api_session, db_connection):
        """Test suppression d'utilisateur"""
        # Créer un utilisateur
        user_data = {'email': f'delete_test_{int(time.time())}@example.com', 'name': 'To Delete'}
        response = api_session.post(f"{APP_URL}/api/users", json=user_data)
        user_id = response.json()['id']
        
        # Supprimer
        response = api_session.delete(f"{APP_URL}/api/users/{user_id}")
        assert response.status_code == 204
        
        # Vérifier absence
        cursor = db_connection.cursor()
        cursor.execute("SELECT id FROM users WHERE id = %s", (user_id,))
        assert cursor.fetchone() is None


class TestTransactions:
    """Tests de comportement transactionnel"""
    
    def test_transaction_commit(self, db_connection):
        """Test commit de transaction"""
        cursor = db_connection.cursor()
        
        # Créer une table de test
        cursor.execute("""
            CREATE TEMPORARY TABLE test_tx (
                id INT AUTO_INCREMENT PRIMARY KEY,
                value INT
            )
        """)
        
        # Transaction avec commit
        db_connection.begin()
        cursor.execute("INSERT INTO test_tx (value) VALUES (100)")
        db_connection.commit()
        
        # Vérifier
        cursor.execute("SELECT value FROM test_tx WHERE value = 100")
        assert cursor.fetchone() is not None
    
    def test_transaction_rollback(self, db_connection):
        """Test rollback de transaction"""
        cursor = db_connection.cursor()
        
        cursor.execute("""
            CREATE TEMPORARY TABLE test_rollback (
                id INT AUTO_INCREMENT PRIMARY KEY,
                value INT
            )
        """)
        
        # Transaction avec rollback
        db_connection.begin()
        cursor.execute("INSERT INTO test_rollback (value) VALUES (999)")
        db_connection.rollback()
        
        # Vérifier absence
        cursor.execute("SELECT value FROM test_rollback WHERE value = 999")
        assert cursor.fetchone() is None


class TestJSONOperations:
    """Tests des opérations JSON"""
    
    def test_json_insert_and_extract(self, db_connection):
        """Test insertion et extraction JSON"""
        cursor = db_connection.cursor()
        
        cursor.execute("""
            CREATE TEMPORARY TABLE test_json (
                id INT AUTO_INCREMENT PRIMARY KEY,
                data JSON
            )
        """)
        
        # Insertion
        json_data = '{"name": "Test", "tags": ["a", "b", "c"]}'
        cursor.execute("INSERT INTO test_json (data) VALUES (?)", (json_data,))
        
        # Extraction
        cursor.execute("SELECT data->>'$.name' AS name FROM test_json")
        assert cursor.fetchone()[0] == "Test"
        
        # Extraction array
        cursor.execute("SELECT JSON_LENGTH(data->'$.tags') FROM test_json")
        assert cursor.fetchone()[0] == 3


class TestPerformanceBaseline:
    """Tests de performance de base"""
    
    def test_simple_query_latency(self, db_connection):
        """Vérifie la latence d'une requête simple"""
        cursor = db_connection.cursor()
        
        start = time.time()
        for _ in range(100):
            cursor.execute("SELECT 1")
            cursor.fetchone()
        end = time.time()
        
        avg_latency_ms = ((end - start) / 100) * 1000
        assert avg_latency_ms < 10, f"Latence trop élevée: {avg_latency_ms}ms"
    
    def test_connection_pool_stress(self, db_connection):
        """Test de stress du pool de connexions"""
        connections = []
        
        try:
            # Ouvrir plusieurs connexions
            for i in range(10):
                conn = mariadb.connect(**DB_CONFIG)
                connections.append(conn)
            
            assert len(connections) == 10
            
        finally:
            # Fermer toutes les connexions
            for conn in connections:
                conn.close()


# Fixtures de setup/teardown
@pytest.fixture(scope="session", autouse=True)
def setup_test_data(db_connection):
    """Setup initial des données de test"""
    cursor = db_connection.cursor()
    
    # Créer la table users si elle n'existe pas
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id INT AUTO_INCREMENT PRIMARY KEY,
            email VARCHAR(255) UNIQUE NOT NULL,
            name VARCHAR(100),
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    """)
    db_connection.commit()
    
    yield
    
    # Cleanup (optionnel)
    # cursor.execute("DROP TABLE IF EXISTS users")
    # db_connection.commit()
```

---

## Phase 4 : Tests de performance

### Benchmarking avec sysbench

```bash
#!/bin/bash
# benchmark_sysbench.sh
# Benchmarking MariaDB avec sysbench

MARIADB_HOST="${1:-localhost}"
MARIADB_PORT="${2:-3306}"
MARIADB_USER="${3:-root}"
MARIADB_PASS="${4:-}"
RESULTS_DIR="./benchmark_results"

mkdir -p $RESULTS_DIR

echo "═══════════════════════════════════════════════════════════════"
echo "   BENCHMARK SYSBENCH - MariaDB"
echo "═══════════════════════════════════════════════════════════════"
echo "Host: $MARIADB_HOST:$MARIADB_PORT"
echo ""

# Configuration sysbench
TABLES=10
TABLE_SIZE=100000
THREADS=8
TIME=60

# Créer la base de test
mariadb -h $MARIADB_HOST -P $MARIADB_PORT -u $MARIADB_USER -p$MARIADB_PASS \
    -e "CREATE DATABASE IF NOT EXISTS sysbench_test"

# 1. Préparation
echo "[1/5] Préparation des données..."
sysbench oltp_read_write \
    --mysql-host=$MARIADB_HOST \
    --mysql-port=$MARIADB_PORT \
    --mysql-user=$MARIADB_USER \
    --mysql-password=$MARIADB_PASS \
    --mysql-db=sysbench_test \
    --tables=$TABLES \
    --table-size=$TABLE_SIZE \
    prepare

# 2. Test OLTP Read/Write
echo ""
echo "[2/5] Test OLTP Read/Write (mixte)..."
sysbench oltp_read_write \
    --mysql-host=$MARIADB_HOST \
    --mysql-port=$MARIADB_PORT \
    --mysql-user=$MARIADB_USER \
    --mysql-password=$MARIADB_PASS \
    --mysql-db=sysbench_test \
    --tables=$TABLES \
    --table-size=$TABLE_SIZE \
    --threads=$THREADS \
    --time=$TIME \
    --report-interval=10 \
    run | tee $RESULTS_DIR/oltp_read_write.txt

# 3. Test Read-Only
echo ""
echo "[3/5] Test Read-Only..."
sysbench oltp_read_only \
    --mysql-host=$MARIADB_HOST \
    --mysql-port=$MARIADB_PORT \
    --mysql-user=$MARIADB_USER \
    --mysql-password=$MARIADB_PASS \
    --mysql-db=sysbench_test \
    --tables=$TABLES \
    --table-size=$TABLE_SIZE \
    --threads=$THREADS \
    --time=$TIME \
    --report-interval=10 \
    run | tee $RESULTS_DIR/oltp_read_only.txt

# 4. Test Write-Only
echo ""
echo "[4/5] Test Write-Only..."
sysbench oltp_write_only \
    --mysql-host=$MARIADB_HOST \
    --mysql-port=$MARIADB_PORT \
    --mysql-user=$MARIADB_USER \
    --mysql-password=$MARIADB_PASS \
    --mysql-db=sysbench_test \
    --tables=$TABLES \
    --table-size=$TABLE_SIZE \
    --threads=$THREADS \
    --time=$TIME \
    --report-interval=10 \
    run | tee $RESULTS_DIR/oltp_write_only.txt

# 5. Cleanup
echo ""
echo "[5/5] Nettoyage..."
sysbench oltp_read_write \
    --mysql-host=$MARIADB_HOST \
    --mysql-port=$MARIADB_PORT \
    --mysql-user=$MARIADB_USER \
    --mysql-password=$MARIADB_PASS \
    --mysql-db=sysbench_test \
    --tables=$TABLES \
    cleanup

# Extraction des métriques clés
echo ""
echo "═══════════════════════════════════════════════════════════════"
echo "   RÉSUMÉ DES RÉSULTATS"
echo "═══════════════════════════════════════════════════════════════"

extract_metrics() {
    local file=$1
    local test_name=$2
    
    tps=$(grep "transactions:" $file | awk '{print $3}' | tr -d '(')
    qps=$(grep "queries:" $file | awk '{print $3}' | tr -d '(')
    lat_avg=$(grep "avg:" $file | head -1 | awk '{print $2}')
    lat_95=$(grep "95th percentile:" $file | awk '{print $3}')
    
    echo "$test_name:"
    echo "  TPS: $tps"
    echo "  QPS: $qps"
    echo "  Latence avg: ${lat_avg}ms"
    echo "  Latence P95: ${lat_95}ms"
    echo ""
}

extract_metrics "$RESULTS_DIR/oltp_read_write.txt" "OLTP Read/Write"
extract_metrics "$RESULTS_DIR/oltp_read_only.txt" "OLTP Read-Only"
extract_metrics "$RESULTS_DIR/oltp_write_only.txt" "OLTP Write-Only"

echo "Résultats détaillés dans: $RESULTS_DIR/"
```

### Test de charge applicatif

```python
# load_test.py
# Test de charge applicatif avec locust

from locust import HttpUser, task, between
import json
import random
import string

class DatabaseUser(HttpUser):
    """Utilisateur simulé pour test de charge"""
    
    wait_time = between(0.5, 2)  # Pause entre requêtes
    
    def on_start(self):
        """Initialisation de l'utilisateur"""
        self.user_ids = []
    
    @task(5)
    def read_users(self):
        """Lecture de la liste des utilisateurs (fréquent)"""
        with self.client.get("/api/users", catch_response=True) as response:
            if response.status_code == 200:
                response.success()
            else:
                response.failure(f"Status code: {response.status_code}")
    
    @task(3)
    def read_single_user(self):
        """Lecture d'un utilisateur spécifique"""
        if self.user_ids:
            user_id = random.choice(self.user_ids)
            with self.client.get(f"/api/users/{user_id}", catch_response=True) as response:
                if response.status_code in [200, 404]:
                    response.success()
                else:
                    response.failure(f"Status code: {response.status_code}")
    
    @task(2)
    def create_user(self):
        """Création d'un utilisateur"""
        random_suffix = ''.join(random.choices(string.ascii_lowercase, k=8))
        user_data = {
            'email': f'loadtest_{random_suffix}@example.com',
            'name': f'Load Test User {random_suffix}'
        }
        
        with self.client.post("/api/users", json=user_data, catch_response=True) as response:
            if response.status_code == 201:
                created_user = response.json()
                self.user_ids.append(created_user['id'])
                response.success()
            else:
                response.failure(f"Status code: {response.status_code}")
    
    @task(1)
    def update_user(self):
        """Mise à jour d'un utilisateur"""
        if self.user_ids:
            user_id = random.choice(self.user_ids)
            update_data = {'name': f'Updated at {random.randint(1, 1000)}'}
            
            with self.client.put(f"/api/users/{user_id}", json=update_data, catch_response=True) as response:
                if response.status_code in [200, 404]:
                    response.success()
                else:
                    response.failure(f"Status code: {response.status_code}")
    
    @task(1)
    def complex_query(self):
        """Requête complexe (agrégation)"""
        with self.client.get("/api/users/stats", catch_response=True) as response:
            if response.status_code == 200:
                response.success()
            else:
                response.failure(f"Status code: {response.status_code}")


# Configuration pour exécution
# locust -f load_test.py --host=http://localhost:8080 --users=100 --spawn-rate=10 --run-time=5m
```

### Comparaison de performance avant/après

```python
# performance_comparison.py
# Comparaison de performance entre deux environnements

import mariadb
import time
import statistics
from dataclasses import dataclass
from typing import List, Dict
import json

@dataclass
class BenchmarkResult:
    """Résultat d'un benchmark"""
    query_name: str
    iterations: int
    min_ms: float
    max_ms: float
    avg_ms: float
    median_ms: float
    p95_ms: float
    p99_ms: float
    std_dev: float


class PerformanceComparator:
    """Compare les performances entre source et cible"""
    
    def __init__(self, source_config: Dict, target_config: Dict):
        self.source_config = source_config
        self.target_config = target_config
    
    def _benchmark_query(self, config: Dict, query: str, iterations: int = 100) -> BenchmarkResult:
        """Benchmark une requête"""
        conn = mariadb.connect(**config)
        cursor = conn.cursor()
        
        # Warm-up
        for _ in range(10):
            cursor.execute(query)
            cursor.fetchall()
        
        # Mesures
        times = []
        for _ in range(iterations):
            start = time.perf_counter()
            cursor.execute(query)
            cursor.fetchall()
            end = time.perf_counter()
            times.append((end - start) * 1000)  # en ms
        
        cursor.close()
        conn.close()
        
        times_sorted = sorted(times)
        
        return BenchmarkResult(
            query_name=query[:50] + "..." if len(query) > 50 else query,
            iterations=iterations,
            min_ms=min(times),
            max_ms=max(times),
            avg_ms=statistics.mean(times),
            median_ms=statistics.median(times),
            p95_ms=times_sorted[int(iterations * 0.95)],
            p99_ms=times_sorted[int(iterations * 0.99)],
            std_dev=statistics.stdev(times) if len(times) > 1 else 0
        )
    
    def compare_query(self, name: str, query: str, iterations: int = 100) -> Dict:
        """Compare une requête entre source et cible"""
        source_result = self._benchmark_query(self.source_config, query, iterations)
        target_result = self._benchmark_query(self.target_config, query, iterations)
        
        # Calcul des deltas
        avg_delta_pct = ((target_result.avg_ms - source_result.avg_ms) / source_result.avg_ms) * 100
        p95_delta_pct = ((target_result.p95_ms - source_result.p95_ms) / source_result.p95_ms) * 100
        
        return {
            'name': name,
            'query': query,
            'source': {
                'avg_ms': source_result.avg_ms,
                'p95_ms': source_result.p95_ms,
                'p99_ms': source_result.p99_ms
            },
            'target': {
                'avg_ms': target_result.avg_ms,
                'p95_ms': target_result.p95_ms,
                'p99_ms': target_result.p99_ms
            },
            'delta': {
                'avg_pct': avg_delta_pct,
                'p95_pct': p95_delta_pct
            },
            'status': 'PASS' if avg_delta_pct < 20 else 'WARNING' if avg_delta_pct < 50 else 'FAIL'
        }
    
    def run_comparison(self, queries: Dict[str, str], iterations: int = 100) -> List[Dict]:
        """Exécute la comparaison pour toutes les requêtes"""
        results = []
        
        print("=" * 80)
        print("COMPARAISON DE PERFORMANCE")
        print("=" * 80)
        print(f"Source: {self.source_config['host']}")
        print(f"Target: {self.target_config['host']}")
        print(f"Iterations par requête: {iterations}")
        print("-" * 80)
        
        for name, query in queries.items():
            print(f"\n[{name}]")
            result = self.compare_query(name, query, iterations)
            results.append(result)
            
            # Affichage
            print(f"  Source: avg={result['source']['avg_ms']:.2f}ms, P95={result['source']['p95_ms']:.2f}ms")
            print(f"  Target: avg={result['target']['avg_ms']:.2f}ms, P95={result['target']['p95_ms']:.2f}ms")
            
            delta = result['delta']['avg_pct']
            status = result['status']
            
            if status == 'PASS':
                print(f"  ✅ Delta: {delta:+.1f}%")
            elif status == 'WARNING':
                print(f"  ⚠️ Delta: {delta:+.1f}% (attention)")
            else:
                print(f"  ❌ Delta: {delta:+.1f}% (dégradation)")
        
        # Résumé
        passed = sum(1 for r in results if r['status'] == 'PASS')
        warnings = sum(1 for r in results if r['status'] == 'WARNING')
        failed = sum(1 for r in results if r['status'] == 'FAIL')
        
        print("\n" + "=" * 80)
        print("RÉSUMÉ")
        print("=" * 80)
        print(f"  ✅ PASS: {passed}")
        print(f"  ⚠️ WARNING: {warnings}")
        print(f"  ❌ FAIL: {failed}")
        
        return results
    
    def export_report(self, results: List[Dict], filepath: str):
        """Exporte le rapport de comparaison"""
        report = {
            'timestamp': __import__('datetime').datetime.now().isoformat(),
            'source': self.source_config['host'],
            'target': self.target_config['host'],
            'results': results
        }
        
        with open(filepath, 'w') as f:
            json.dump(report, f, indent=2)
        
        print(f"\nRapport exporté: {filepath}")


# Exemple d'utilisation
if __name__ == '__main__':
    source_config = {
        'host': 'mysql-source',
        'port': 3306,
        'user': 'bench_user',
        'password': 'bench_password',
        'database': 'benchmark_db'
    }
    
    target_config = {
        'host': 'mariadb-target',
        'port': 3306,
        'user': 'bench_user',
        'password': 'bench_password',
        'database': 'benchmark_db'
    }
    
    # Requêtes à benchmarker
    queries = {
        'simple_select': "SELECT 1",
        'count_users': "SELECT COUNT(*) FROM users",
        'join_query': """
            SELECT u.name, COUNT(o.id) as order_count
            FROM users u
            LEFT JOIN orders o ON u.id = o.user_id
            GROUP BY u.id
            LIMIT 100
        """,
        'complex_aggregation': """
            SELECT 
                DATE(created_at) as date,
                COUNT(*) as total,
                SUM(amount) as revenue
            FROM orders
            WHERE created_at > DATE_SUB(NOW(), INTERVAL 30 DAY)
            GROUP BY DATE(created_at)
            ORDER BY date
        """
    }
    
    comparator = PerformanceComparator(source_config, target_config)
    results = comparator.run_comparison(queries, iterations=100)
    comparator.export_report(results, 'performance_report.json')
```

---

## Phase 5 : Validation en production

### Stratégie Canary

```
Déploiement Canary
━━━━━━━━━━━━━━━━━━

Phase 1: 1% du trafic
┌─────────────────────────────────────────────────────────────┐
│                      Load Balancer                          │
└─────────────────────────────────────────────────────────────┘
                              │
               ┌──────────────┴──────────────┐
               │ 99%                    1%   │
               ▼                             ▼
      ┌─────────────────┐           ┌─────────────────┐
      │  MySQL (Source) │           │ MariaDB (Canary)│
      │                 │           │                 │
      └─────────────────┘           └─────────────────┘

Phase 2: 10% du trafic
Phase 3: 25% du trafic
Phase 4: 50% du trafic
Phase 5: 100% du trafic (cutover complet)
```

### Configuration ProxySQL pour canary

```sql
-- Configuration ProxySQL pour déploiement canary

-- Ajout des serveurs
INSERT INTO mysql_servers (hostgroup_id, hostname, port, weight, comment) VALUES
    (10, 'mysql-source', 3306, 99, 'MySQL Source - Primary'),
    (10, 'mariadb-canary', 3306, 1, 'MariaDB Canary - 1%');

-- Règle de routage pour le groupe de lecture
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply) VALUES
    (100, 1, '^SELECT', 10, 1);

-- Charger la configuration
LOAD MYSQL SERVERS TO RUNTIME;
LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
SAVE MYSQL QUERY RULES TO DISK;

-- Pour augmenter le pourcentage canary (ex: 10%)
UPDATE mysql_servers SET weight = 90 WHERE hostname = 'mysql-source';
UPDATE mysql_servers SET weight = 10 WHERE hostname = 'mariadb-canary';
LOAD MYSQL SERVERS TO RUNTIME;
```

### Monitoring du canary

```python
# canary_monitor.py
# Monitoring du déploiement canary

import time
import mariadb
from datetime import datetime
from dataclasses import dataclass
from typing import List, Dict
import statistics

@dataclass
class CanaryMetrics:
    """Métriques du canary"""
    timestamp: datetime
    source_qps: float
    canary_qps: float
    source_error_rate: float
    canary_error_rate: float
    source_latency_p95: float
    canary_latency_p95: float


class CanaryMonitor:
    """Moniteur de déploiement canary"""
    
    def __init__(self, source_config: Dict, canary_config: Dict):
        self.source_config = source_config
        self.canary_config = canary_config
        self.metrics_history: List[CanaryMetrics] = []
        self.alerts = []
    
    def collect_metrics(self) -> CanaryMetrics:
        """Collecte les métriques des deux environnements"""
        # Dans un cas réel, ces métriques viendraient de Prometheus/Grafana
        # Ici, simulation avec requêtes directes
        
        def get_server_metrics(config: Dict) -> Dict:
            conn = mariadb.connect(**config)
            cursor = conn.cursor()
            
            # Questions (queries)
            cursor.execute("SHOW GLOBAL STATUS LIKE 'Questions'")
            questions = int(cursor.fetchone()[1])
            
            # Erreurs
            cursor.execute("SHOW GLOBAL STATUS LIKE 'Connection_errors_max_connections'")
            errors = int(cursor.fetchone()[1])
            
            conn.close()
            
            return {'questions': questions, 'errors': errors}
        
        source_metrics = get_server_metrics(self.source_config)
        canary_metrics = get_server_metrics(self.canary_config)
        
        return CanaryMetrics(
            timestamp=datetime.now(),
            source_qps=source_metrics['questions'],  # Simplifié
            canary_qps=canary_metrics['questions'],
            source_error_rate=0.0,  # À calculer
            canary_error_rate=0.0,
            source_latency_p95=0.0,  # À mesurer
            canary_latency_p95=0.0
        )
    
    def check_thresholds(self, metrics: CanaryMetrics) -> List[str]:
        """Vérifie les seuils d'alerte"""
        alerts = []
        
        # Seuil d'erreurs
        if metrics.canary_error_rate > 0.01:  # 1%
            alerts.append(f"ALERT: Canary error rate {metrics.canary_error_rate:.2%} > 1%")
        
        # Seuil de latence
        if metrics.canary_latency_p95 > metrics.source_latency_p95 * 1.5:
            alerts.append(f"ALERT: Canary latency 50% higher than source")
        
        return alerts
    
    def monitor_loop(self, interval_seconds: int = 60, duration_minutes: int = 60):
        """Boucle de monitoring"""
        print("=" * 60)
        print("CANARY MONITORING")
        print("=" * 60)
        print(f"Interval: {interval_seconds}s")
        print(f"Duration: {duration_minutes}min")
        print("-" * 60)
        
        end_time = time.time() + (duration_minutes * 60)
        
        while time.time() < end_time:
            metrics = self.collect_metrics()
            self.metrics_history.append(metrics)
            
            alerts = self.check_thresholds(metrics)
            
            # Affichage
            print(f"\n[{metrics.timestamp.strftime('%H:%M:%S')}]")
            print(f"  Source QPS: {metrics.source_qps:.1f}")
            print(f"  Canary QPS: {metrics.canary_qps:.1f}")
            
            if alerts:
                self.alerts.extend(alerts)
                for alert in alerts:
                    print(f"  🚨 {alert}")
            else:
                print("  ✅ All metrics within thresholds")
            
            time.sleep(interval_seconds)
        
        self.generate_report()
    
    def generate_report(self):
        """Génère le rapport final"""
        print("\n" + "=" * 60)
        print("CANARY MONITORING REPORT")
        print("=" * 60)
        
        if not self.metrics_history:
            print("No metrics collected")
            return
        
        print(f"Duration: {len(self.metrics_history)} samples")
        print(f"Alerts triggered: {len(self.alerts)}")
        
        if self.alerts:
            print("\nAlerts:")
            for alert in self.alerts[:10]:  # Top 10
                print(f"  - {alert}")
        
        # Recommandation
        print("\nRecommendation:")
        if len(self.alerts) == 0:
            print("  ✅ SAFE TO PROCEED - Increase canary percentage")
        elif len(self.alerts) < 5:
            print("  ⚠️ CAUTION - Review alerts before proceeding")
        else:
            print("  ❌ ROLLBACK RECOMMENDED - Too many alerts")


# Utilisation
if __name__ == '__main__':
    source_config = {
        'host': 'mysql-source',
        'port': 3306,
        'user': 'monitor',
        'password': 'monitor_pass',
        'database': 'mysql'
    }
    
    canary_config = {
        'host': 'mariadb-canary',
        'port': 3306,
        'user': 'monitor',
        'password': 'monitor_pass',
        'database': 'mysql'
    }
    
    monitor = CanaryMonitor(source_config, canary_config)
    monitor.monitor_loop(interval_seconds=30, duration_minutes=10)
```

---

## Critères de go/no-go

### Checklist de validation finale

```markdown
## CHECKLIST GO/NO-GO - Migration MariaDB

### Tests Techniques
- [ ] Connexion établie avec tous les connecteurs
- [ ] Version MariaDB 11.8 confirmée
- [ ] TLS fonctionnel
- [ ] Charset utf8mb4 vérifié

### Tests Fonctionnels
- [ ] 100% des requêtes critiques passent
- [ ] Checksums données source = cible
- [ ] Procédures stockées fonctionnelles
- [ ] Triggers actifs et testés

### Tests d'Intégration
- [ ] API endpoints fonctionnels
- [ ] Transactions ACID vérifiées
- [ ] Opérations CRUD OK
- [ ] Batch jobs exécutés

### Tests de Performance
- [ ] Latence P95 < 120% de la source
- [ ] TPS >= 90% de la source
- [ ] Aucune requête 5x plus lente
- [ ] Test de charge validé

### Validation Production
- [ ] Canary 1% stable 24h
- [ ] Canary 10% stable 24h
- [ ] Monitoring configuré
- [ ] Alerting opérationnel
- [ ] Plan de rollback testé

### Documentation
- [ ] Runbook de migration
- [ ] Procédure de rollback
- [ ] Contacts d'urgence
- [ ] Communication planifiée

### Approbations
- [ ] DBA Lead
- [ ] Tech Lead
- [ ] Product Owner
- [ ] Ops Manager
```

### Matrice de décision

| Critère | Seuil GO | Seuil NO-GO |
|---------|----------|-------------|
| Tests fonctionnels | 100% pass | < 100% pass |
| Delta performance (avg) | < +20% | > +50% |
| Delta performance (P95) | < +30% | > +100% |
| Erreurs canary | < 0.1% | > 1% |
| Durée canary stable | > 24h | < 4h |
| Alertes critiques | 0 | > 0 |

---

## ✅ Points clés à retenir

- Les tests de compatibilité suivent une **pyramide** : technique → fonctionnel → intégration → performance → production
- Chaque phase a un **objectif distinct** et des outils adaptés
- La **comparaison source/cible** avec checksums garantit l'intégrité des données
- Les **benchmarks sysbench** fournissent une baseline de performance objective
- Le **déploiement canary** permet une validation progressive en production réelle
- Définissez des **critères go/no-go clairs** avant de commencer les tests
- **Automatisez** au maximum pour garantir la reproductibilité
- **Documentez** tous les résultats pour l'audit et le post-mortem

---

## 🔗 Ressources et références

- [📖 sysbench Documentation](https://github.com/akopytov/sysbench)
- [📖 Locust Load Testing](https://locust.io/)
- [📖 pytest Documentation](https://docs.pytest.org/)
- [📖 ProxySQL Documentation](https://proxysql.com/documentation/)
- [📖 MariaDB Performance Schema](https://mariadb.com/kb/en/performance-schema/)

---

## ➡️ Section suivante

**[19.7 Rollback et contingence](./07-rollback-contingence.md)** : Nous aborderons la conception de plans de rollback robustes, les procédures d'urgence, la gestion des données créées post-migration, et les stratégies de communication de crise.

⏭️ [Rollback et contingence](/19-migration-compatibilite/07-rollback-contingence.md)
