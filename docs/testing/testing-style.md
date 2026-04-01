# Testing Style Guide

## RSpec Request Testing Style

This codebase follows a **comprehensive, behavior-driven RSpec request testing pattern** with these key characteristics:

### Structure & Organization
- **Nested context blocks** for different scenarios (valid/invalid credentials, missing fields, etc.)
- **Clear subject definitions** at the top of each describe block
- **Before/after hooks** for test data setup and cleanup
- **Let blocks** for parameter definitions that can be overridden in contexts

### Testing Approach
- **Schema validation integration** - validates both request and response against OpenAPI schemas
- **HTTP status code verification** - explicit status code checking
- **Response structure validation** - checks JSON response format and content
- **Database state verification** - confirms side effects (like user creation)
- **Error handling coverage** - tests various error scenarios comprehensively

### Key Patterns
- **Parameterized testing** - uses `let` blocks to define request bodies that can be customized per context
- **Explicit cleanup** - removes test data in `after` blocks to prevent test pollution
- **Schema-driven validation** - leverages OpenAPI schemas to validate request/response formats
- **Symbolized response checking** - uses `json_response_symbolized` helper for easier hash comparison
- **Change matchers** - uses `expect { }.to change { }` to verify database mutations

### Style Characteristics
- **Descriptive context names** that clearly indicate what's being tested
- **Comprehensive edge case coverage** (missing fields, invalid values, duplicates)
- **Explicit expectations** rather than implicit assertions
- **Clean separation** between setup, execution, and verification phases

This style emphasizes **completeness, clarity, and maintainability** while ensuring both functional correctness and API contract compliance.

### Example Structure
```ruby
RSpec.describe "Resource", type: :request do
  let(:schema) { JSONSchemer.openapi(YAML.load_file('docs/api/v1/api.yml')) }

  describe 'POST /v1/resource' do
    subject { post '/v1/resource', params: body, as: :json }
    let(:body) { {} }

    before(:all) do
      # Setup test data
    end

    after(:all) do
      # Cleanup test data
    end

    context 'with valid parameters' do
      let(:body) { { valid: 'data' } }

      it 'succeeds and returns expected response' do
        expect(schema.schema('RequestSchema').valid?(body)).to be true

        subject

        expect(response).to have_http_status(:ok)
        expect(schema.schema('ResponseSchema').valid?(json_response)).to be true
        expect(json_response_symbolized).to eq(expected_response)
      end
    end

    context 'with invalid parameters' do
      context 'when required field is missing' do
        let(:body) { { optional: 'data' } }

        it 'returns appropriate error' do
          expect(schema.schema('RequestSchema').valid?(body)).to be false

          subject

          expect(response).to have_http_status(:bad_request)
          expect(schema.schema('ErrorResponse').valid?(json_response)).to be true
          expect(json_response_symbolized).to eq(expected_error)
        end
      end
    end
  end
end
```
