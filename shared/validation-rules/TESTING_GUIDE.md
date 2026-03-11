# Guia de Testes Unitários — Validação de Value Objects

Este documento é a referência para uma IA (ou desenvolvedor) implementar testes unitários que validem cada Value Object do domínio `social-care`. Todos os testes devem garantir que as regras de negócio dos YAMLs em `contracts/shared/validation-rules/` estejam corretamente implementadas.

---

## Stack e Framework de Testes

```swift
import Testing                    // swift-testing (NÃO usar XCTest)
import Foundation
@testable import social_care_s   // módulo do serviço
```

- **Framework**: `swift-testing` (Swift 6.2)
- **Assertions**: `#expect()` — NUNCA `XCTAssert`
- **Structs** como suítes — NUNCA classes
- **async/await** em todos os testes que tocam actors ou handlers

---

## Estrutura de um Arquivo de Teste

```swift
import Testing
import Foundation
@testable import social_care_s

@Suite("NomeDoVO - Validações")
struct NomeDoVOTests {

    @Test("Deve criar com valores válidos")
    func validCreation() throws {
        let vo = try NomeDoVO(/* params válidos */)
        #expect(vo.property == expectedValue)
    }

    @Test("Deve rejeitar quando [condição inválida]")
    func invalidCondition() {
        #expect(throws: NomeDoVOError.specificError) {
            try NomeDoVO(/* params inválidos */)
        }
    }
}
```

### Convenções obrigatórias

| Item | Regra |
|------|-------|
| Nome da struct | `{NomeDoVO}Tests` (ex: `CPFTests`, `AddressTests`) |
| `@Suite` | Descrição em português: `"CPF - Validações"` |
| `@Test` | Descrição em português: `"Deve rejeitar CPF com todos dígitos iguais"` |
| Nome do método | camelCase descritivo em inglês: `rejectAllSameDigits()` |
| Assertions | Apenas `#expect()` |
| Erros síncronos | `#expect(throws: ErrorType.case) { try VO(...) }` |
| Erros assíncronos | `await #expect(throws: ErrorType.self) { try await handler.handle(...) }` |

---

## Categorias de Teste por VO

Para **cada** Value Object, implementar testes nestas categorias:

### 1. Happy Path (criação válida)
Verificar que o VO é criado com sucesso e os campos são normalizados corretamente.

### 2. Normalização
Verificar que trim, collapse_whitespace, uppercase, lowercase, remoção de caracteres especiais funcionam.

### 3. Rejeição por campo vazio/nulo
Para cada campo `required: true`, testar com string vazia, apenas espaços, e nil.

### 4. Rejeição por formato inválido
Testar regex, length, charset violations.

### 5. Rejeição por regra de negócio
Testar validações custom (checksum, ranges, cross-field).

### 6. Igualdade e Hashability
Verificar `Equatable` e `Hashable` quando relevante.

---

## Testes por Value Object

### Kernel: CPF

```swift
@Suite("CPF - Validações")
struct CPFTests {

    // ── Happy Path ──────────────────────────────────────────────

    @Test("Deve criar CPF válido a partir de string formatada")
    func validFormatted() throws {
        let cpf = try CPF("529.982.247-25")
        #expect(cpf.value == "52998224725")
        #expect(cpf.formatted == "529.982.247-25")
    }

    @Test("Deve criar CPF válido a partir de string sem formatação")
    func validUnformatted() throws {
        let cpf = try CPF("52998224725")
        #expect(cpf.value == "52998224725")
    }

    // ── Normalização ────────────────────────────────────────────

    @Test("Deve sanitizar espaços e caracteres antes de validar")
    func sanitization() throws {
        let cpf = try CPF("  529.982.247-25  ")
        #expect(cpf.value == "52998224725")
    }

    // ── Rejeições ───────────────────────────────────────────────

    @Test("Deve rejeitar CPF vazio")
    func rejectEmpty() {
        #expect(throws: CPFError.empty) {
            try CPF("")
        }
    }

    @Test("Deve rejeitar CPF apenas com espaços")
    func rejectWhitespace() {
        #expect(throws: CPFError.empty) {
            try CPF("   ")
        }
    }

    @Test("Deve rejeitar CPF com caracteres inválidos")
    func rejectInvalidChars() {
        #expect(throws: CPFError.invalidCharacters) {
            try CPF("529ABC24725")
        }
    }

    @Test("Deve rejeitar CPF com menos de 11 dígitos")
    func rejectTooShort() {
        #expect(throws: CPFError.invalidLength) {
            try CPF("1234567890")
        }
    }

    @Test("Deve rejeitar CPF com mais de 11 dígitos")
    func rejectTooLong() {
        #expect(throws: CPFError.invalidLength) {
            try CPF("123456789012")
        }
    }

    @Test("Deve rejeitar CPF com todos os dígitos iguais")
    func rejectAllSameDigits() {
        // Testar todos: 00000000000, 11111111111, ..., 99999999999
        for digit in 0...9 {
            let repeated = String(repeating: "\(digit)", count: 11)
            #expect(throws: CPFError.repeatedDigits) {
                try CPF(repeated)
            }
        }
    }

    @Test("Deve rejeitar CPF com dígitos verificadores inválidos")
    func rejectInvalidCheckDigits() {
        #expect(throws: CPFError.invalidCheckDigits) {
            try CPF("52998224700")  // últimos 2 dígitos errados
        }
    }

    // ── Derived ─────────────────────────────────────────────────

    @Test("Deve derivar região fiscal a partir do dígito 8")
    func fiscalRegion() throws {
        let cpf = try CPF("52998224725")  // dígito 8 = '4'
        #expect(cpf.fiscalRegion == "AL, PB, PE, RN")
    }
}
```

### Kernel: NIS

```swift
@Suite("NIS - Validações")
struct NISTests {

    @Test("Deve criar NIS válido com 11 dígitos")
    func validNIS() throws {
        let nis = try NIS("12345678901")
        #expect(nis.value == "12345678901")
    }

    @Test("Deve sanitizar removendo não-dígitos")
    func sanitization() throws {
        let nis = try NIS("123.4567.890-1")
        #expect(nis.value == "12345678901")
    }

    @Test("Deve rejeitar NIS vazio")
    func rejectEmpty() {
        #expect(throws: NISError.empty) { try NIS("") }
    }

    @Test("Deve rejeitar NIS com quantidade errada de dígitos")
    func rejectWrongLength() {
        #expect(throws: NISError.invalidLength) { try NIS("1234567890") }    // 10
        #expect(throws: NISError.invalidLength) { try NIS("123456789012") }  // 12
    }
}
```

### Kernel: CEP

```swift
@Suite("CEP - Validações")
struct CEPTests {

    @Test("Deve criar CEP válido")
    func validCEP() throws {
        let cep = try CEP("01310-100")
        #expect(cep.value == "01310100")
        #expect(cep.formatted == "01310-100")
    }

    @Test("Deve sanitizar hífens e espaços")
    func sanitization() throws {
        let cep = try CEP("  01310 100  ")
        #expect(cep.value == "01310100")
    }

    @Test("Deve rejeitar CEP vazio")
    func rejectEmpty() {
        #expect(throws: CEPError.empty) { try CEP("") }
    }

    @Test("Deve rejeitar CEP com caracteres inválidos")
    func rejectInvalidChars() {
        #expect(throws: CEPError.invalidCharacters) { try CEP("0131A100") }
    }

    @Test("Deve rejeitar CEP com tamanho errado")
    func rejectWrongLength() {
        #expect(throws: CEPError.invalidLength) { try CEP("0131010") }   // 7 dígitos
        #expect(throws: CEPError.invalidLength) { try CEP("013101001") } // 9 dígitos
    }

    @Test("Deve rejeitar CEP fora de faixa válida de UF")
    func rejectOutOfRange() {
        #expect(throws: CEPError.invalidRange) { try CEP("00000000") }
    }

    @Test("Deve aceitar CEP de cada UF")
    func acceptAllStates() throws {
        // Testar pelo menos um CEP de cada faixa
        let samples: [String: String] = [
            "SP": "01310100", "RJ": "20040020", "MG": "30130000",
            "BA": "40020000", "RS": "90010000", "PR": "80010000",
        ]
        for (_, cep) in samples {
            _ = try CEP(cep)
        }
    }
}
```

### Kernel: PersonId / ProfessionalId / LookupId / PatientId / AppointmentId / ReferralId / ViolationReportId

Todos os IDs baseados em UUID seguem o **mesmo padrão de teste**:

```swift
@Suite("PersonId - Validações")
struct PersonIdTests {

    @Test("Deve criar a partir de UUID válido")
    func validUUID() throws {
        let id = try PersonId("550e8400-e29b-41d4-a716-446655440000")
        #expect(id.value == "550e8400-e29b-41d4-a716-446655440000")
    }

    @Test("Deve normalizar para lowercase")
    func lowercaseNormalization() throws {
        let id = try PersonId("550E8400-E29B-41D4-A716-446655440000")
        #expect(id.value == "550e8400-e29b-41d4-a716-446655440000")
    }

    @Test("Deve trimmar espaços")
    func trimWhitespace() throws {
        let id = try PersonId("  550e8400-e29b-41d4-a716-446655440000  ")
        #expect(id.value == "550e8400-e29b-41d4-a716-446655440000")
    }

    @Test("Deve gerar automaticamente quando sem argumento")
    func autoGenerate() {
        let id = PersonId()
        #expect(!id.value.isEmpty)
    }

    @Test("Deve rejeitar formato inválido")
    func rejectInvalid() {
        #expect(throws: PIDError.invalidFormat) { try PersonId("not-a-uuid") }
    }

    @Test("Deve rejeitar string vazia")
    func rejectEmpty() {
        #expect(throws: PIDError.invalidFormat) { try PersonId("") }
    }
}
```

> **Repetir esse template** para: `ProfessionalId` (erro: `ProfessionalIdError`), `LookupId` (`LookupIdError`), `PatientId` (`PatientIdError`), `AppointmentId` (`AppointmentIdError`), `ReferralId` (`ReferralIdError`), `ViolationReportId` (`ViolationReportIdError`).

### Kernel: RGDocument

```swift
@Suite("RGDocument - Validações")
struct RGDocumentTests {

    // ── Happy Path ──────────────────────────────────────────────

    @Test("Deve criar RG válido com dígito verificador numérico")
    func validNumericCheckDigit() throws {
        let rg = try RGDocument(
            number: "12345678-9",  // ajustar para um RG com check digit válido
            issuingState: "SP",
            issuingAgency: "SSP",
            issueDate: Date(timeIntervalSince1970: 0),
            now: Date()
        )
        #expect(rg.issuingState == "SP")
        #expect(rg.issuingAgency == "SSP")
    }

    // ── Normalização ────────────────────────────────────────────

    @Test("Deve normalizar número removendo pontos e hífens e convertendo para uppercase")
    func numberNormalization() throws {
        // Verificar que "12.345.678-X" vira "12345678X"
    }

    @Test("Deve normalizar UF para uppercase")
    func stateUppercase() throws {
        let rg = try RGDocument(number: "...", issuingState: "sp", issuingAgency: "SSP", issueDate: ..., now: ...)
        #expect(rg.issuingState == "SP")
    }

    @Test("Deve normalizar órgão emissor colapsando espaços e uppercasing")
    func agencyNormalization() throws {
        let rg = try RGDocument(number: "...", issuingState: "SP", issuingAgency: "  ssp   sp  ", issueDate: ..., now: ...)
        #expect(rg.issuingAgency == "SSP SP")
    }

    // ── Rejeições ───────────────────────────────────────────────

    @Test("Deve rejeitar número vazio")
    func rejectEmptyNumber() {
        #expect(throws: RGDocumentError.emptyNumber) {
            try RGDocument(number: "", issuingState: "SP", issuingAgency: "SSP", issueDate: ..., now: ...)
        }
    }

    @Test("Deve rejeitar número fora do formato 8+1 dígitos")
    func rejectInvalidFormat() {
        #expect(throws: RGDocumentError.invalidFormat) {
            try RGDocument(number: "1234567", issuingState: "SP", issuingAgency: "SSP", issueDate: ..., now: ...)
        }
    }

    @Test("Deve rejeitar dígito verificador incorreto")
    func rejectInvalidCheckDigit() {
        #expect(throws: RGDocumentError.invalidCheckDigit) {
            try RGDocument(number: "123456780", issuingState: "SP", issuingAgency: "SSP", issueDate: ..., now: ...)
        }
    }

    @Test("Deve rejeitar UF inválida")
    func rejectInvalidState() {
        #expect(throws: RGDocumentError.invalidState) {
            try RGDocument(number: "...", issuingState: "XX", issuingAgency: "SSP", issueDate: ..., now: ...)
        }
    }

    @Test("Deve rejeitar órgão emissor vazio")
    func rejectEmptyAgency() {
        #expect(throws: RGDocumentError.emptyAgency) {
            try RGDocument(number: "...", issuingState: "SP", issuingAgency: "", issueDate: ..., now: ...)
        }
    }

    @Test("Deve rejeitar data de emissão no futuro")
    func rejectFutureDate() {
        let futureDate = Date().addingTimeInterval(86400 * 365) // 1 ano no futuro
        #expect(throws: RGDocumentError.futureDateNotAllowed) {
            try RGDocument(number: "...", issuingState: "SP", issuingAgency: "SSP", issueDate: futureDate, now: Date())
        }
    }
}
```

### Kernel: Address

```swift
@Suite("Address - Validações")
struct AddressTests {

    @Test("Deve criar endereço válido mínimo (sem CEP)")
    func validMinimal() throws {
        let addr = try Address(
            isShelter: false,
            residenceLocation: .urbano,
            state: "SP",
            city: "São Paulo"
        )
        #expect(addr.state == "SP")
        #expect(addr.city == "São Paulo")
        #expect(addr.cep == nil)
    }

    @Test("Deve criar endereço completo com CEP")
    func validComplete() throws {
        let addr = try Address(
            cep: try CEP("01310100"),
            isShelter: true,
            residenceLocation: .rural,
            street: "Rua Teste",
            neighborhood: "Centro",
            number: "123",
            complement: "Apto 4",
            state: "SP",
            city: "São Paulo"
        )
        #expect(addr.isShelter == true)
        #expect(addr.residenceLocation == .rural)
    }

    @Test("Deve normalizar campos de texto colapsando espaços")
    func textNormalization() throws {
        let addr = try Address(
            isShelter: false,
            residenceLocation: .urbano,
            street: "  Rua   do   Teste  ",
            state: "sp",
            city: "  São   Paulo  "
        )
        #expect(addr.street == "Rua do Teste")
        #expect(addr.city == "São Paulo")
        #expect(addr.state == "SP")
    }

    @Test("Deve rejeitar UF vazia")
    func rejectEmptyState() {
        #expect(throws: AddressError.emptyState) {
            try Address(isShelter: false, residenceLocation: .urbano, state: "", city: "City")
        }
    }

    @Test("Deve rejeitar UF inválida")
    func rejectInvalidState() {
        #expect(throws: AddressError.invalidState) {
            try Address(isShelter: false, residenceLocation: .urbano, state: "XX", city: "City")
        }
    }

    @Test("Deve rejeitar cidade vazia")
    func rejectEmptyCity() {
        #expect(throws: AddressError.emptyCity) {
            try Address(isShelter: false, residenceLocation: .urbano, state: "SP", city: "")
        }
    }

    @Test("Deve rejeitar CEP inválido quando fornecido")
    func rejectInvalidCep() {
        #expect(throws: AddressError.invalidCep) {
            try Address(cep: try CEP("00000000"), isShelter: false, residenceLocation: .urbano, state: "SP", city: "SP")
        }
    }
}
```

### Registry: PersonalData

```swift
@Suite("PersonalData - Validações")
struct PersonalDataTests {

    @Test("Deve criar com dados válidos completos")
    func validComplete() throws {
        let pd = try PersonalData(
            firstName: "Maria",
            lastName: "Silva",
            motherName: "Ana Silva",
            nationality: "Brasileira",
            sex: .feminino,
            birthDate: try TimeStamp(iso: "1990-01-01T00:00:00.000Z"),
            now: .now
        )
        #expect(pd.firstName == "Maria")
        #expect(pd.sex == .feminino)
    }

    @Test("Deve normalizar nomes colapsando espaços")
    func nameNormalization() throws {
        let pd = try PersonalData(
            firstName: "  Maria   Luísa  ",
            lastName: "  da   Silva  ",
            motherName: "  Ana   Maria  ",
            nationality: "  Brasileira  ",
            sex: .feminino,
            birthDate: try TimeStamp(iso: "1990-01-01T00:00:00.000Z"),
            now: .now
        )
        #expect(pd.firstName == "Maria Luísa")
        #expect(pd.lastName == "da Silva")
        #expect(pd.motherName == "Ana Maria")
    }

    @Test("Deve tratar socialName vazio como nil")
    func socialNameEmptyToNil() throws {
        let pd = try PersonalData(
            firstName: "Maria", lastName: "Silva", motherName: "Ana",
            nationality: "Brasileira", sex: .feminino,
            socialName: "   ",
            birthDate: try TimeStamp(iso: "1990-01-01T00:00:00.000Z"),
            now: .now
        )
        #expect(pd.socialName == nil)
    }

    @Test("Deve tratar phone vazio como nil")
    func phoneEmptyToNil() throws {
        let pd = try PersonalData(
            firstName: "Maria", lastName: "Silva", motherName: "Ana",
            nationality: "Brasileira", sex: .feminino,
            birthDate: try TimeStamp(iso: "1990-01-01T00:00:00.000Z"),
            phone: "  ",
            now: .now
        )
        #expect(pd.phone == nil)
    }

    @Test("Deve rejeitar nome vazio")
    func rejectEmptyFirstName() {
        #expect(throws: PersonalDataError.firstNameEmpty) {
            try PersonalData(firstName: "", lastName: "Silva", motherName: "Ana",
                nationality: "BR", sex: .feminino,
                birthDate: try TimeStamp(iso: "1990-01-01T00:00:00.000Z"), now: .now)
        }
    }

    @Test("Deve rejeitar sobrenome vazio")
    func rejectEmptyLastName() {
        #expect(throws: PersonalDataError.lastNameEmpty) {
            try PersonalData(firstName: "Maria", lastName: "  ", motherName: "Ana",
                nationality: "BR", sex: .feminino,
                birthDate: try TimeStamp(iso: "1990-01-01T00:00:00.000Z"), now: .now)
        }
    }

    @Test("Deve rejeitar nome da mãe vazio")
    func rejectEmptyMotherName() {
        #expect(throws: PersonalDataError.motherNameEmpty) {
            try PersonalData(firstName: "Maria", lastName: "Silva", motherName: "",
                nationality: "BR", sex: .feminino,
                birthDate: try TimeStamp(iso: "1990-01-01T00:00:00.000Z"), now: .now)
        }
    }

    @Test("Deve rejeitar nacionalidade vazia")
    func rejectEmptyNationality() {
        #expect(throws: PersonalDataError.nationalityEmpty) {
            try PersonalData(firstName: "Maria", lastName: "Silva", motherName: "Ana",
                nationality: "", sex: .feminino,
                birthDate: try TimeStamp(iso: "1990-01-01T00:00:00.000Z"), now: .now)
        }
    }

    @Test("Deve rejeitar data de nascimento no futuro")
    func rejectFutureBirthDate() {
        let future = Date().addingTimeInterval(86400 * 365)
        #expect(throws: PersonalDataError.birthDateInFuture) {
            try PersonalData(firstName: "Maria", lastName: "Silva", motherName: "Ana",
                nationality: "BR", sex: .feminino,
                birthDate: TimeStamp(future), now: .now)
        }
    }
}
```

### Registry: CivilDocuments

```swift
@Suite("CivilDocuments - Validações")
struct CivilDocumentsTests {

    @Test("Deve criar com apenas CPF")
    func onlyCPF() throws {
        let docs = try CivilDocuments(cpf: try CPF("52998224725"))
        #expect(docs.cpf != nil)
        #expect(docs.nis == nil)
        #expect(docs.rgDocument == nil)
    }

    @Test("Deve criar com apenas NIS")
    func onlyNIS() throws {
        let docs = try CivilDocuments(nis: try NIS("12345678901"))
        #expect(docs.nis != nil)
    }

    @Test("Deve criar com todos os documentos")
    func allDocuments() throws {
        let docs = try CivilDocuments(
            cpf: try CPF("52998224725"),
            nis: try NIS("12345678901"),
            rgDocument: try RGDocument(...)
        )
        #expect(docs.cpf != nil)
        #expect(docs.nis != nil)
        #expect(docs.rgDocument != nil)
    }

    @Test("Deve rejeitar quando nenhum documento é fornecido")
    func rejectAllEmpty() {
        #expect(throws: CivilDocumentsError.atLeastOneDocumentRequired) {
            try CivilDocuments()
        }
    }
}
```

### Registry: SocialIdentity

```swift
@Suite("SocialIdentity - Validações")
struct SocialIdentityTests {

    @Test("Deve criar com tipo normal sem descrição")
    func normalType() throws {
        let si = try SocialIdentity(
            typeId: try LookupId(UUID().uuidString),
            isOtherType: false
        )
        #expect(si.otherDescription == nil)
    }

    @Test("Deve criar com tipo 'Outras' e descrição preenchida")
    func otherTypeWithDescription() throws {
        let si = try SocialIdentity(
            typeId: try LookupId(UUID().uuidString),
            isOtherType: true,
            otherDescription: "Comunidade quilombola"
        )
        #expect(si.otherDescription == "Comunidade quilombola")
    }

    @Test("Deve rejeitar tipo 'Outras' sem descrição")
    func rejectOtherWithoutDescription() {
        #expect(throws: SocialIdentityError.descriptionRequiredForOtherType) {
            try SocialIdentity(
                typeId: try LookupId(UUID().uuidString),
                isOtherType: true,
                otherDescription: nil
            )
        }
    }

    @Test("Deve rejeitar tipo 'Outras' com descrição apenas espaços")
    func rejectOtherWithWhitespaceDescription() {
        #expect(throws: SocialIdentityError.descriptionRequiredForOtherType) {
            try SocialIdentity(
                typeId: try LookupId(UUID().uuidString),
                isOtherType: true,
                otherDescription: "   "
            )
        }
    }
}
```

### Assessment: HousingCondition

```swift
@Suite("HousingCondition - Validações")
struct HousingConditionTests {

    @Test("Deve criar com valores válidos")
    func validCreation() throws {
        let hc = try HousingCondition(
            type: .owned, wallMaterial: .masonry,
            numberOfRooms: 5, numberOfBedrooms: 2, numberOfBathrooms: 1,
            waterSupply: .publicNetwork, hasPipedWater: true,
            electricityAccess: .meteredConnection,
            sewageDisposal: .publicSewer, wasteCollection: .directCollection,
            accessibilityLevel: .fullyAccessible,
            isInGeographicRiskArea: false, hasDifficultAccess: false,
            isInSocialConflictArea: false, hasDiagnosticObservations: false
        )
        #expect(hc.numberOfRooms == 5)
    }

    @Test("Deve rejeitar número negativo de cômodos")
    func rejectNegativeRooms() {
        #expect(throws: HousingConditionError.negativeRooms) {
            try HousingCondition(type: .owned, wallMaterial: .masonry,
                numberOfRooms: -1, numberOfBedrooms: 0, numberOfBathrooms: 0, ...)
        }
    }

    @Test("Deve rejeitar número negativo de quartos")
    func rejectNegativeBedrooms() {
        #expect(throws: HousingConditionError.negativeBedrooms) {
            try HousingCondition(..., numberOfRooms: 5, numberOfBedrooms: -1, numberOfBathrooms: 0, ...)
        }
    }

    @Test("Deve rejeitar número negativo de banheiros")
    func rejectNegativeBathrooms() {
        #expect(throws: HousingConditionError.negativeBathrooms) {
            try HousingCondition(..., numberOfRooms: 5, numberOfBedrooms: 2, numberOfBathrooms: -1, ...)
        }
    }

    @Test("Deve rejeitar quartos excedendo total de cômodos")
    func rejectBedroomsExceedRooms() {
        #expect(throws: HousingConditionError.bedroomsExceedRooms) {
            try HousingCondition(..., numberOfRooms: 3, numberOfBedrooms: 5, numberOfBathrooms: 1, ...)
        }
    }

    @Test("Deve aceitar quartos igual ao total de cômodos (limite)")
    func acceptBedroomsEqualToRooms() throws {
        let hc = try HousingCondition(..., numberOfRooms: 3, numberOfBedrooms: 3, numberOfBathrooms: 0, ...)
        #expect(hc.numberOfBedrooms == 3)
    }
}
```

### Assessment: SocioEconomicSituation

```swift
@Suite("SocioEconomicSituation - Validações")
struct SocioEconomicSituationTests {

    @Test("Deve criar com dados válidos sem benefícios")
    func validWithoutBenefits() throws {
        let ses = try SocioEconomicSituation(
            totalFamilyIncome: 2000.0,
            incomePerCapita: 500.0,
            receivesSocialBenefit: false,
            socialBenefits: try SocialBenefitsCollection([]),
            mainSourceOfIncome: "Trabalho formal",
            hasUnemployed: false
        )
        #expect(ses.totalFamilyIncome == 2000.0)
    }

    @Test("Deve rejeitar renda negativa")
    func rejectNegativeIncome() {
        #expect(throws: SocioEconomicSituationError.negativeFamilyIncome) {
            try SocioEconomicSituation(totalFamilyIncome: -100.0, ...)
        }
    }

    @Test("Deve rejeitar renda per capita negativa")
    func rejectNegativePerCapita() {
        #expect(throws: SocioEconomicSituationError.negativeIncomePerCapita) {
            try SocioEconomicSituation(totalFamilyIncome: 1000.0, incomePerCapita: -1.0, ...)
        }
    }

    @Test("Deve rejeitar per capita maior que renda total")
    func rejectPerCapitaExceedsTotal() {
        #expect(throws: SocioEconomicSituationError.inconsistentIncomePerCapita) {
            try SocioEconomicSituation(totalFamilyIncome: 1000.0, incomePerCapita: 1500.0, ...)
        }
    }

    @Test("Deve rejeitar fonte de renda vazia")
    func rejectEmptySource() {
        #expect(throws: SocioEconomicSituationError.emptyMainSourceOfIncome) {
            try SocioEconomicSituation(..., mainSourceOfIncome: "  ", ...)
        }
    }

    @Test("Deve rejeitar flag false com benefícios presentes")
    func rejectFlagFalseWithBenefits() {
        #expect(throws: SocioEconomicSituationError.inconsistentSocialBenefit) {
            try SocioEconomicSituation(
                ..., receivesSocialBenefit: false,
                socialBenefits: try SocialBenefitsCollection([
                    try SocialBenefit(benefitName: "BPC", amount: 1412.0, beneficiaryId: PersonId())
                ]), ...
            )
        }
    }

    @Test("Deve rejeitar flag true sem benefícios")
    func rejectFlagTrueWithoutBenefits() {
        #expect(throws: SocioEconomicSituationError.missingSocialBenefits) {
            try SocioEconomicSituation(
                ..., receivesSocialBenefit: true,
                socialBenefits: try SocialBenefitsCollection([]), ...
            )
        }
    }
}
```

### Assessment: SocialBenefit e SocialBenefitsCollection

```swift
@Suite("SocialBenefit - Validações")
struct SocialBenefitTests {

    @Test("Deve criar benefício válido")
    func valid() throws {
        let b = try SocialBenefit(benefitName: "BPC", amount: 1412.0, beneficiaryId: PersonId())
        #expect(b.benefitName == "BPC")
        #expect(b.amount == 1412.0)
    }

    @Test("Deve normalizar nome colapsando espaços")
    func nameNormalization() throws {
        let b = try SocialBenefit(benefitName: "  Bolsa   Família  ", amount: 600.0, beneficiaryId: PersonId())
        #expect(b.benefitName == "Bolsa Família")
    }

    @Test("Deve rejeitar nome vazio")
    func rejectEmptyName() {
        #expect(throws: SocialBenefitError.benefitNameEmpty) {
            try SocialBenefit(benefitName: "", amount: 100.0, beneficiaryId: PersonId())
        }
    }

    @Test("Deve rejeitar valor zero")
    func rejectZeroAmount() {
        #expect(throws: SocialBenefitError.amountInvalid) {
            try SocialBenefit(benefitName: "BPC", amount: 0, beneficiaryId: PersonId())
        }
    }

    @Test("Deve rejeitar valor negativo")
    func rejectNegativeAmount() {
        #expect(throws: SocialBenefitError.amountInvalid) {
            try SocialBenefit(benefitName: "BPC", amount: -50.0, beneficiaryId: PersonId())
        }
    }
}

@Suite("SocialBenefitsCollection - Validações")
struct SocialBenefitsCollectionTests {

    @Test("Deve criar coleção vazia")
    func emptyCollection() throws {
        let col = try SocialBenefitsCollection([])
        #expect(col.isEmpty == true)
        #expect(col.count == 0)
        #expect(col.totalAmount == 0)
    }

    @Test("Deve calcular totalAmount corretamente")
    func totalAmountCalculation() throws {
        let col = try SocialBenefitsCollection([
            try SocialBenefit(benefitName: "BPC", amount: 1412.0, beneficiaryId: PersonId()),
            try SocialBenefit(benefitName: "Bolsa Família", amount: 600.0, beneficiaryId: PersonId()),
        ])
        #expect(col.totalAmount == 2012.0)
        #expect(col.count == 2)
    }

    @Test("Deve rejeitar benefícios com nome duplicado")
    func rejectDuplicateNames() {
        #expect(throws: SocialBenefitsCollectionError.duplicateBenefitNotAllowed) {
            try SocialBenefitsCollection([
                try SocialBenefit(benefitName: "BPC", amount: 1412.0, beneficiaryId: PersonId()),
                try SocialBenefit(benefitName: "BPC", amount: 800.0, beneficiaryId: PersonId()),
            ])
        }
    }
}
```

### Assessment: WorkAndIncome (item de renda)

```swift
@Suite("WorkIncomeVO - Validações")
struct WorkIncomeVOTests {

    @Test("Deve criar com renda válida")
    func valid() throws {
        let wi = try WorkIncomeVO(memberId: PersonId(), monthlyAmount: 1500.0)
        #expect(wi.monthlyAmount == 1500.0)
    }

    @Test("Deve aceitar renda zero")
    func acceptZero() throws {
        let wi = try WorkIncomeVO(memberId: PersonId(), monthlyAmount: 0)
        #expect(wi.monthlyAmount == 0)
    }

    @Test("Deve rejeitar renda negativa")
    func rejectNegative() {
        #expect(throws: WorkIncomeError.negativeMonthlyAmount) {
            try WorkIncomeVO(memberId: PersonId(), monthlyAmount: -100.0)
        }
    }
}
```

### Assessment: CommunitySupportNetwork

```swift
@Suite("CommunitySupportNetwork - Validações")
struct CommunitySupportNetworkTests {

    @Test("Deve criar com dados válidos")
    func valid() throws {
        let csn = try CommunitySupportNetwork(
            hasRelativeSupport: true, hasNeighborSupport: false,
            familyConflicts: "Conflito sobre guarda",
            patientParticipatesInGroups: true, familyParticipatesInGroups: false,
            patientHasAccessToLeisure: true, facesDiscrimination: false
        )
        #expect(csn.familyConflicts == "Conflito sobre guarda")
    }

    @Test("Deve rejeitar conflitos apenas whitespace")
    func rejectWhitespace() {
        #expect(throws: CommunitySupportNetworkError.familyConflictsWhitespace) {
            try CommunitySupportNetwork(..., familyConflicts: "   ", ...)
        }
    }

    @Test("Deve rejeitar conflitos com mais de 300 caracteres")
    func rejectTooLong() {
        let longText = String(repeating: "a", count: 301)
        #expect(throws: CommunitySupportNetworkError.familyConflictsTooLong) {
            try CommunitySupportNetwork(..., familyConflicts: longText, ...)
        }
    }

    @Test("Deve aceitar conflitos com exatamente 300 caracteres (limite)")
    func acceptExactLimit() throws {
        let exactText = String(repeating: "a", count: 300)
        let csn = try CommunitySupportNetwork(..., familyConflicts: exactText, ...)
        #expect(csn.familyConflicts.count == 300)
    }
}
```

### Assessment: SocialHealthSummary

```swift
@Suite("SocialHealthSummary - Validações")
struct SocialHealthSummaryTests {

    @Test("Deve criar com lista de dependências válidas")
    func valid() throws {
        let shs = try SocialHealthSummary(
            requiresConstantCare: true, hasMobilityImpairment: false,
            functionalDependencies: ["Alimentação", "Higiene"],
            hasRelevantDrugTherapy: true
        )
        #expect(shs.functionalDependencies.count == 2)
    }

    @Test("Deve deduplica itens repetidos preservando ordem")
    func deduplication() throws {
        let shs = try SocialHealthSummary(
            ..., functionalDependencies: ["Alimentação", "Higiene", "Alimentação"], ...
        )
        #expect(shs.functionalDependencies == ["Alimentação", "Higiene"])
    }

    @Test("Deve rejeitar item vazio na lista")
    func rejectEmptyItem() {
        #expect(throws: SocialHealthSummaryError.functionalDependenciesEmpty) {
            try SocialHealthSummary(
                ..., functionalDependencies: ["Alimentação", "", "Higiene"], ...
            )
        }
    }

    @Test("Deve rejeitar item com apenas espaços na lista")
    func rejectWhitespaceItem() {
        #expect(throws: SocialHealthSummaryError.functionalDependenciesEmpty) {
            try SocialHealthSummary(
                ..., functionalDependencies: ["Alimentação", "   "], ...
            )
        }
    }
}
```

### Care: ICDCode

```swift
@Suite("ICDCode - Validações")
struct ICDCodeTests {

    @Test("Deve criar CID válido e normalizar para uppercase")
    func valid() throws {
        let icd = try ICDCode("b201")
        #expect(icd.value.uppercased() == icd.value || icd.value == "B20.1" || icd.value == "B201")
    }

    @Test("Deve inserir ponto automaticamente antes do último caractere (auto-dot)")
    func autoDot() throws {
        let icd = try ICDCode("A169")
        #expect(icd.value == "A16.9")
    }

    @Test("Deve manter ponto existente")
    func existingDot() throws {
        let icd = try ICDCode("A16.9")
        #expect(icd.value == "A16.9")
    }

    @Test("Deve rejeitar código vazio")
    func rejectEmpty() {
        #expect(throws: ICDCodeError.empty) {
            try ICDCode("")
        }
    }

    @Test("Dois CIDs com e sem ponto devem ser equivalentes")
    func equivalence() throws {
        let a = try ICDCode("A16.9")
        let b = try ICDCode("A169")
        #expect(a.isEquivalent(to: b))
    }
}
```

### Care: Diagnosis

```swift
@Suite("Diagnosis - Validações")
struct DiagnosisTests {

    @Test("Deve criar diagnóstico válido")
    func valid() throws {
        let diag = try Diagnosis(
            id: try ICDCode("B201"),
            date: TimeStamp.now,
            description: "Doença pelo HIV",
            now: .now
        )
        #expect(diag.description == "Doença pelo HIV")
    }

    @Test("Deve rejeitar data no futuro")
    func rejectFutureDate() {
        let future = Date().addingTimeInterval(86400 * 30)
        #expect(throws: DiagnosisError.dateInFuture) {
            try Diagnosis(id: try ICDCode("B201"), date: TimeStamp(future), description: "Test", now: .now)
        }
    }

    @Test("Deve rejeitar descrição vazia")
    func rejectEmptyDescription() {
        #expect(throws: DiagnosisError.emptyDescription) {
            try Diagnosis(id: try ICDCode("B201"), date: .now, description: "  ", now: .now)
        }
    }
}
```

### Care: IngressInfo

```swift
@Suite("IngressInfo - Validações")
struct IngressInfoTests {

    @Test("Deve criar com dados válidos")
    func valid() throws {
        let info = try IngressInfo(
            ingressTypeId: try LookupId(UUID().uuidString),
            serviceReason: "Encaminhamento CRAS"
        )
        #expect(info.serviceReason == "Encaminhamento CRAS")
    }

    @Test("Deve rejeitar motivo vazio")
    func rejectEmptyReason() {
        #expect(throws: IngressInfoError.emptyServiceReason) {
            try IngressInfo(
                ingressTypeId: try LookupId(UUID().uuidString),
                serviceReason: "  "
            )
        }
    }
}
```

### Care: SocialCareAppointment

```swift
@Suite("SocialCareAppointment - Validações")
struct SocialCareAppointmentTests {

    @Test("Deve criar atendimento válido")
    func valid() throws {
        let apt = try SocialCareAppointment(
            date: .now,
            professionalInChargeId: ProfessionalId(),
            type: .homeVisit,
            summary: "Visita realizada",
            actionPlan: nil,
            now: .now
        )
        #expect(apt.summary == "Visita realizada")
    }

    @Test("Deve rejeitar data no futuro")
    func rejectFutureDate() {
        let future = Date().addingTimeInterval(86400)
        #expect(throws: SocialCareAppointmentError.dateInFuture) {
            try SocialCareAppointment(date: TimeStamp(future), ..., now: .now)
        }
    }

    @Test("Deve rejeitar quando summary e actionPlan são ambos vazios")
    func rejectBothEmpty() {
        #expect(throws: SocialCareAppointmentError.atLeastOneNarrative) {
            try SocialCareAppointment(date: .now, ..., summary: nil, actionPlan: nil, now: .now)
        }
    }

    @Test("Deve rejeitar summary com mais de 500 caracteres")
    func rejectLongSummary() {
        let long = String(repeating: "x", count: 501)
        #expect(throws: SocialCareAppointmentError.summaryTooLong) {
            try SocialCareAppointment(date: .now, ..., summary: long, actionPlan: nil, now: .now)
        }
    }

    @Test("Deve rejeitar actionPlan com mais de 2000 caracteres")
    func rejectLongActionPlan() {
        let long = String(repeating: "x", count: 2001)
        #expect(throws: SocialCareAppointmentError.actionPlanTooLong) {
            try SocialCareAppointment(date: .now, ..., summary: nil, actionPlan: long, now: .now)
        }
    }
}
```

### Protection: Referral

```swift
@Suite("Referral - Validações")
struct ReferralTests {

    @Test("Deve criar encaminhamento válido com status PENDING")
    func valid() throws {
        let ref = try Referral(
            date: .now,
            requestingProfessionalId: ProfessionalId(),
            referredPersonId: PersonId(),
            destinationService: .cras,
            reason: "Situação de vulnerabilidade",
            now: .now
        )
        #expect(ref.status == .pending)
    }

    @Test("Deve transitar de PENDING para COMPLETED")
    func completeTransition() throws {
        var ref = try Referral(date: .now, ..., now: .now)
        try ref.complete()
        #expect(ref.status == .completed)
    }

    @Test("Deve transitar de PENDING para CANCELLED")
    func cancelTransition() throws {
        var ref = try Referral(date: .now, ..., now: .now)
        try ref.cancel()
        #expect(ref.status == .cancelled)
    }

    @Test("Deve rejeitar transição de COMPLETED para qualquer estado")
    func rejectFromCompleted() throws {
        var ref = try Referral(date: .now, ..., now: .now)
        try ref.complete()
        #expect(throws: ReferralError.invalidStatusTransition) { try ref.complete() }
        #expect(throws: ReferralError.invalidStatusTransition) { try ref.cancel() }
    }

    @Test("Deve rejeitar transição de CANCELLED para qualquer estado")
    func rejectFromCancelled() throws {
        var ref = try Referral(date: .now, ..., now: .now)
        try ref.cancel()
        #expect(throws: ReferralError.invalidStatusTransition) { try ref.complete() }
        #expect(throws: ReferralError.invalidStatusTransition) { try ref.cancel() }
    }

    @Test("Deve rejeitar data no futuro")
    func rejectFutureDate() {
        #expect(throws: ReferralError.dateInFuture) {
            try Referral(date: TimeStamp(Date().addingTimeInterval(86400)), ..., now: .now)
        }
    }

    @Test("Deve rejeitar motivo vazio")
    func rejectEmptyReason() {
        #expect(throws: ReferralError.emptyReason) {
            try Referral(date: .now, ..., reason: "  ", now: .now)
        }
    }
}
```

### Protection: RightsViolationReport

```swift
@Suite("RightsViolationReport - Validações")
struct RightsViolationReportTests {

    @Test("Deve criar notificação válida")
    func valid() throws {
        let report = try RightsViolationReport(
            reportDate: .now,
            victimId: PersonId(),
            violationType: .neglect,
            descriptionOfFact: "Criança sem alimentação adequada",
            now: .now
        )
        #expect(report.violationType == .neglect)
    }

    @Test("Deve rejeitar data da notificação no futuro")
    func rejectFutureReportDate() {
        #expect(throws: RightsViolationReportError.reportDateInFuture) {
            try RightsViolationReport(
                reportDate: TimeStamp(Date().addingTimeInterval(86400)),
                ..., now: .now
            )
        }
    }

    @Test("Deve rejeitar data do incidente posterior à data da notificação")
    func rejectIncidentAfterReport() {
        let reportDate = try TimeStamp(iso: "2025-01-01T00:00:00.000Z")
        let incidentDate = try TimeStamp(iso: "2025-06-01T00:00:00.000Z")
        #expect(throws: RightsViolationReportError.incidentAfterReport) {
            try RightsViolationReport(
                reportDate: reportDate, incidentDate: incidentDate,
                ..., now: TimeStamp(Date())
            )
        }
    }

    @Test("Deve rejeitar descrição vazia")
    func rejectEmptyDescription() {
        #expect(throws: RightsViolationReportError.emptyDescription) {
            try RightsViolationReport(
                reportDate: .now, ..., descriptionOfFact: "  ", now: .now
            )
        }
    }

    @Test("Deve aceitar incidentDate nil")
    func acceptNilIncidentDate() throws {
        let report = try RightsViolationReport(
            reportDate: .now, incidentDate: nil, ..., now: .now
        )
        #expect(report.incidentDate == nil)
    }
}
```

### Protection: PlacementHistory

```swift
@Suite("PlacementHistory - Validações")
struct PlacementHistoryTests {

    @Test("Deve criar registro com datas válidas")
    func validRegistry() throws {
        let start = try TimeStamp(iso: "2025-01-01T00:00:00.000Z")
        let end = try TimeStamp(iso: "2025-06-01T00:00:00.000Z")
        let registry = try PlacementHistory.PlacementRegistry(
            memberId: PersonId(),
            startDate: start,
            endDate: end,
            reason: "Acolhimento institucional"
        )
        #expect(registry.endDate != nil)
    }

    @Test("Deve aceitar endDate nil (acolhimento em andamento)")
    func acceptNilEndDate() throws {
        let registry = try PlacementHistory.PlacementRegistry(
            memberId: PersonId(),
            startDate: .now,
            endDate: nil,
            reason: "Em andamento"
        )
        #expect(registry.endDate == nil)
    }

    @Test("Deve rejeitar endDate anterior a startDate")
    func rejectEndBeforeStart() {
        let start = try TimeStamp(iso: "2025-06-01T00:00:00.000Z")
        let end = try TimeStamp(iso: "2025-01-01T00:00:00.000Z")
        #expect(throws: PlacementError.invalidDateRange) {
            try PlacementHistory.PlacementRegistry(
                memberId: PersonId(),
                startDate: start,
                endDate: end,
                reason: "Teste"
            )
        }
    }
}
```

---

## Testes de Analytics (Indicadores Calculados)

### Housing Analytics

```swift
@Suite("HousingAnalyticsService - Cálculos")
struct HousingAnalyticsTests {

    @Test("Deve calcular densidade corretamente")
    func density() {
        let density = HousingAnalyticsService.density(forMembers: 6, inBedrooms: 2)
        #expect(density == 3.0)
    }

    @Test("Deve usar mínimo 1 para evitar divisão por zero")
    func densityMinimumOne() {
        let density = HousingAnalyticsService.density(forMembers: 0, inBedrooms: 0)
        #expect(density == 1.0)  // max(0,1) / max(0,1)
    }

    @Test("Deve identificar superlotação quando density > 3.0")
    func overcrowded() {
        #expect(HousingAnalyticsService.isOvercrowded(members: 7, bedrooms: 2) == true)
    }

    @Test("Deve identificar NÃO superlotação quando density <= 3.0")
    func notOvercrowded() {
        #expect(HousingAnalyticsService.isOvercrowded(members: 6, bedrooms: 2) == false)
    }
}
```

### Financial Analytics

```swift
@Suite("FinancialAnalyticsService - Cálculos")
struct FinancialAnalyticsTests {

    @Test("Deve calcular indicadores financeiros corretamente")
    func indicators() {
        let result = FinancialAnalyticsService.calculate(
            workIncomes: [1000.0, 500.0],
            socialBenefitAmounts: [600.0],
            memberCount: 4
        )
        #expect(result.totalWorkIncome == 1500.0)
        #expect(result.perCapitaWorkIncome == 375.0)       // 1500 / 4
        #expect(result.totalGlobalIncome == 2100.0)        // 1500 + 600
        #expect(result.perCapitaGlobalIncome == 525.0)     // 2100 / 4
    }

    @Test("Deve usar memberCount mínimo 1")
    func minimumMemberCount() {
        let result = FinancialAnalyticsService.calculate(
            workIncomes: [1000.0],
            socialBenefitAmounts: [],
            memberCount: 0
        )
        #expect(result.perCapitaWorkIncome == 1000.0)  // 1000 / max(0,1)
    }
}
```

### Education Analytics

```swift
@Suite("EducationAnalyticsService - Vulnerabilidades")
struct EducationAnalyticsTests {

    @Test("Deve detectar criança 6-14 fora da escola")
    func childOutOfSchool() {
        let members = [
            EducationalMember(personId: PersonId(), birthDate: /* 10 anos */, attendsSchool: false, canReadWrite: true)
        ]
        let report = EducationAnalyticsService.calculateVulnerabilities(members: members, at: .now)
        #expect(report.count(.notInSchool, .range6to14) == 1)
    }

    @Test("Deve detectar analfabetismo em adulto 18-59")
    func adultIlliteracy() {
        let members = [
            EducationalMember(personId: PersonId(), birthDate: /* 30 anos */, attendsSchool: false, canReadWrite: false)
        ]
        let report = EducationAnalyticsService.calculateVulnerabilities(members: members, at: .now)
        #expect(report.count(.illiteracy, .range18to59) == 1)
    }
}
```

---

## Checklist de Cobertura por VO

Use esta tabela para rastrear quais testes foram implementados:

| VO | Happy Path | Normalização | Campo vazio | Formato | Regra negócio | Igualdade | Total mínimo |
|----|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| CPF | ◻ | ◻ | ◻ | ◻ | ◻ (checksum, repeated) | ◻ | 8+ |
| NIS | ◻ | ◻ | ◻ | ◻ | — | ◻ | 4+ |
| CEP | ◻ | ◻ | ◻ | ◻ | ◻ (range) | ◻ | 7+ |
| PersonId | ◻ | ◻ | ◻ | ◻ | — | ◻ | 5+ |
| ProfessionalId | ◻ | ◻ | ◻ | ◻ | — | ◻ | 5+ |
| LookupId | ◻ | ◻ | ◻ | ◻ | — | ◻ | 5+ |
| PatientId | ◻ | ◻ | ◻ | ◻ | — | ◻ | 5+ |
| RGDocument | ◻ | ◻ | ◻ | ◻ | ◻ (check digit) | — | 8+ |
| Address | ◻ | ◻ | ◻ | — | ◻ (UF, CEP) | — | 7+ |
| TimeStamp | ◻ | — | ◻ | ◻ (ISO8601) | ◻ (isSameDay, years) | ◻ | 5+ |
| PersonalData | ◻ | ◻ | ◻ (×5 campos) | — | ◻ (birthDate futuro) | — | 9+ |
| CivilDocuments | ◻ | — | — | — | ◻ (≥1 doc) | — | 3+ |
| SocialIdentity | ◻ | — | — | — | ◻ (other→desc) | — | 4+ |
| FamilyMember | ◻ | — | — | — | ◻ (dedup docs) | ◻ (by personId) | 3+ |
| HousingCondition | ◻ | — | — | — | ◻ (≥0, bed≤rooms) | — | 6+ |
| HealthStatus | ◻ | — | — | — | — (sem validação) | — | 1+ |
| EducationalStatus | ◻ | — | — | — | — (sem validação) | — | 1+ |
| SocioEconomicSit. | ◻ | — | ◻ | — | ◻ (flag↔benefits, per capita) | — | 7+ |
| SocialBenefit | ◻ | ◻ | ◻ | — | ◻ (amount > 0) | — | 5+ |
| SocialBenefitsColl | ◻ | — | — | — | ◻ (no duplicates) | — | 3+ |
| WorkIncomeVO | ◻ | — | — | — | ◻ (≥0) | — | 3+ |
| CommunitySuppNet | ◻ | — | — | — | ◻ (whitespace, max300) | — | 4+ |
| SocialHealthSum | ◻ | — | — | — | ◻ (dedup, no empty) | — | 4+ |
| ICDCode | ◻ | ◻ | ◻ | — | ◻ (auto-dot, equiv) | — | 5+ |
| Diagnosis | ◻ | — | ◻ | — | ◻ (date futuro) | — | 3+ |
| IngressInfo | ◻ | — | ◻ | — | — | — | 2+ |
| AppointmentId | ◻ | ◻ | ◻ | ◻ | — | ◻ | 5+ |
| SCA (Appointment) | ◻ | — | — | — | ◻ (≥1 narrative, max len) | — | 5+ |
| Referral | ◻ | — | ◻ | — | ◻ (state machine ×4) | — | 7+ |
| RightsViolation | ◻ | — | ◻ | — | ◻ (incident≤report) | — | 5+ |
| PlacementRegistry | ◻ | — | — | — | ◻ (end≥start) | — | 3+ |
| **Analytics** | | | | | | | |
| HousingAnalytics | — | — | — | — | ◻ (density, overcrowded) | — | 4+ |
| FinancialAnalytics | — | — | — | — | ◻ (indicadores) | — | 2+ |
| EducationAnalytics | — | — | — | — | ◻ (vulnerabilities) | — | 2+ |
| FamilyAnalytics | — | — | — | — | ◻ (age buckets) | — | 2+ |

**Total mínimo estimado: ~150 testes**

---

## Dicas de Implementação

1. **Boundary values**: sempre testar o limite exato (300 chars OK, 301 falha; 0 OK, -1 falha; bedrooms == rooms OK, bedrooms == rooms+1 falha).

2. **Normalização antes de validação**: testar que `"  Maria   Silva  "` é normalizado para `"Maria Silva"` ANTES da checagem de vazio.

3. **null_if_empty**: campos como `socialName` e `phone` devem virar `nil` quando string é vazia após normalização.

4. **Datas**: usar `TimeStamp.now` como referência e `Date().addingTimeInterval(86400)` para futuro.

5. **UUIDs nos testes**: usar `UUID().uuidString` para gerar IDs válidos ou constantes fixas para reprodutibilidade.

6. **Testes de cada enum value**: para enums como `DestinationService`, `ViolationType`, `AppointmentType`, testar criação com cada valor válido e rejeição de valor inválido.

7. **Concorrência**: para actors (use cases), testar com `async let` para verificar isolamento.
