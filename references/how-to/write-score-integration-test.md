# Write SCORE integration test

This document explains how to write SCORE integration test using `IconIntegrateTestBase`.

### Purpose

Understand how to write SCORE integration test

### Prerequisite

* [SCORE Overview](../../python-score/overview.md)
* [T-Bears Overview](../../tbears/overview.md) 
* [ICON Python SDK](../../icon-sdks/python-sdk/)

### How to Write SCORE Integration Test Code

The SCORE integration test code works as follows.

1. Deploy the SCORE to be tested
2. Create an ICON JSON-RPC API request for the SCORE API you want to test
3. If necessary, sign an ICON JSON-RPC API request
4. Invoke an ICON JSON-RPC API request and get the result
5. Check the result

#### Packages and modules

**ICON Python SDK**

You can create and sign an ICON JSON-RPC API request using the ICON Python SDK

```python
# create key wallet
self._test = KeyWallet.create()

# Generates an instance of transaction for deploying SCORE.
transaction = DeployTransactionBuilder() \
    .from_(self._test.get_address()) \
    .to(to) \
    .step_limit(100_000_000_000) \
    .nid(3) \
    .nonce(100) \
    .content_type("application/zip") \
    .content(gen_deploy_data_content(self.SCORE_PROJECT)) \
    .build()

# Returns the signed transaction object having a signature
signed_transaction = SignedTransaction(transaction, self._test)
```

**IconIntegrateTestBase in T-Bears**

Every SCORE integration test class must inherit `IconIntegrateTestBase`. `IconIntegrateTestBase` class provides three functions

1. Support Python unittest 1. You can write and run the test method with prefix 'test\_' 2. You can initialize and finalize the test by override setUp and tearDown method
2. Emulate ICON service for test 1. Initialize ICON service and confirm genesis block 2. Create accounts for test 1. self.\_test1 : Account with 1,000,000 ICX 2. self.\_wallet\_array\[\] : 10 empty accounts in list
3. Provide API for SCORE integration test 1. process\_transaction\(\) Invoke transaction and return transaction result 2. process\_call\(\) Calls SCORE's external function which is read-only and returns result

#### examples

You can get the source code with `tbears init score_test ScoreTest` command.

**score\_test.py**

```python
from iconservice import *

TAG = 'ScoreTest'

class ScoreTest(IconScoreBase):

    def __init__(self, db: IconScoreDatabase) -> None:
        super().__init__(db)

    def on_install(self) -> None:
        super().on_install()

    def on_update(self) -> None:
        super().on_update()

    @external(readonly=True)
    def hello(self) -> str:
        Logger.debug(f'Hello, world!', TAG)
        return "Hello"
```

**score\_tests/test\_score\_test.py**

```python
import os

from iconsdk.builder.transaction_builder import DeployTransactionBuilder
from iconsdk.builder.call_builder import CallBuilder
from iconsdk.libs.in_memory_zip import gen_deploy_data_content
from iconsdk.signed_transaction import SignedTransaction

from tbears.libs.icon_integrate_test import IconIntegrateTestBase, SCORE_INSTALL_ADDRESS

DIR_PATH = os.path.abspath(os.path.dirname(__file__))


class TestScoreTest(IconIntegrateTestBase):
    TEST_HTTP_ENDPOINT_URI_V3 = "http://127.0.0.1:9000/api/v3"
    SCORE_PROJECT= os.path.abspath(os.path.join(DIR_PATH, '..'))

    def setUp(self):
        super().setUp()

        self.icon_service = None
        # If you want to send request to network, uncomment next line and set self.TEST_HTTP_ENDPOINT_URI_V3
        # self.icon_service = IconService(HTTPProvider(self.TEST_HTTP_ENDPOINT_URI_V3))

        # deploy SCORE
        self._score_address = self._deploy_score()['scoreAddress']

    def _deploy_score(self, to: str = SCORE_INSTALL_ADDRESS) -> dict:
        # Generates an instance of transaction for deploying SCORE.
        transaction = DeployTransactionBuilder() \
            .from_(self._test1.get_address()) \
            .to(to) \
            .step_limit(100_000_000_000) \
            .nid(3) \
            .nonce(100) \
            .content_type("application/zip") \
            .content(gen_deploy_data_content(self.SCORE_PROJECT)) \
            .build()

        # Returns the signed transaction object having a signature
        signed_transaction = SignedTransaction(transaction, self._test1)

        # process the transaction in local
        tx_result = self._process_transaction(signed_transaction)

        # check transaction result
        self.assertTrue('status' in tx_result)
        self.assertEqual(1, tx_result['status'])
        self.assertTrue('scoreAddress' in tx_result)

        return tx_result

    def test_score_update(self):
        # update SCORE
        tx_result = self._deploy_score(self._score_address)

        self.assertEqual(self._score_address, tx_result['scoreAddress'])

    def test_call_hello(self):
        # Generates a call instance using the CallBuilder
        call = CallBuilder().from_(self._test1.get_address()) \
            .to(self._score_address) \
            .method("hello") \
            .build()

        # Sends the call request
        response = self._process_call(call, self.icon_service)

        # check call result
        self.assertEqual("Hello", response)
```

**Run test code**

```bash
$ tbears test score_test
..
----------------------------------------------------------------------
Ran 2 tests in 0.172s

OK
```

### References

* [ICON Python SDK](../../icon-sdks/python-sdk/)
* [ICON SCORE samples](../../python-score/sample-scores/)

