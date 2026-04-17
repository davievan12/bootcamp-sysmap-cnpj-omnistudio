# Test Plan

## Comprehensive Test Plan

### Test Scenarios

1. **Cache Hit**  
   **Inputs:**  
   - Valid data retrieved from cache  
   **Expected Outputs:**  
   - Data returned successfully without hitting the API  
   **Validation Steps:**  
   - Verify that the data is the same as what was previously stored in the cache.  
   **Evidence:**  
   - Screenshots of cache storage and result returned.

2. **Cache Miss**  
   **Inputs:**  
   - Valid data not present in cache  
   **Expected Outputs:**  
   - API called to retrieve data  
   **Validation Steps:**  
   - Check logs to ensure API call was made and data was retrieved from the API.  
   **Evidence:**  
   - Screenshots of logs showing API interaction.

3. **ForceAPI**  
   **Inputs:**  
   - Forced retrieval from API regardless of cache  
   **Expected Outputs:**  
   - Fresh data retrieved from API  
   **Validation Steps:**  
   - Ensure that the cache was ignored and the API was called.  
   **Evidence:**  
   - Screenshots of API response vs cached data.

4. **CNPJ Inválido**  
   **Inputs:**  
   - Invalid CNPJ provided  
   **Expected Outputs:**  
   - Error message indicating CNPJ is invalid  
   **Validation Steps:**  
   - Verify that error message matches expected format and content for invalid CNPJ.  
   **Evidence:**  
   - Screenshots of error message displayed to the user.

5. **CEP Auto-complete**  
   **Inputs:**  
   - Partial CEP input  
   **Expected Outputs:**  
   - Suggestions for completing CEP  
   **Validation Steps:**  
   - Check that suggestions correspond to existing, valid CEPs.  
   **Evidence:**  
   - Screenshots of suggestions shown in the UI.

6. **CNPJ Não Encontrado**  
   **Inputs:**  
   - Non-existent CNPJ provided  
   **Expected Outputs:**  
   - Message indicating CNPJ was not found  
   **Validation Steps:**  
   - Verify that the appropriate message is displayed for not found cases.  
   **Evidence:**  
   - Screenshots of the message displayed after search.

---

### Summary  
This test plan covers the essential test scenarios for the application, ensuring comprehensive testing and validation for proper functioning of features.