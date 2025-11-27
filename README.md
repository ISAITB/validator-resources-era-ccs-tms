# CCSTMS Validator

[![License: EUPL-1.2](https://img.shields.io/badge/License-EUPL--1.2-blue.svg)](https://joinup.ec.europa.eu/collection/eupl/eupl-text-eupl-12)
[![ITB Compatible](https://img.shields.io/badge/ITB-Compatible-green.svg)](https://www.itb.ec.europa.eu/)
[![ERA Ontology](https://img.shields.io/badge/ERA-Extended%20Ontology-orange.svg)](https://www.era.europa.eu/)

A SHACL-based RDF validator for the **Command, Control & Signalling, and Traffic Management System (CCSTMS)** domain, built on the European Union Agency for Railways (ERA) extended ontology. This validator enables stakeholders to validate RDF data against extended ERA ontology covering multiple railway domains with a specific focus on ETCS Level2 without light signal

## Overview

The CCSTMS Validator is developed as part of the collaboration between the **European Union Agency for Railways (ERA)** and **Europe's Rail Joint Undertaking (ERJU)** to support interoperability and standardization in European railway systems. It leverages the [Interoperability Test Bed (ITB)](https://www.itb.ec.europa.eu/) infrastructure for reliable, standards-compliant validation services.

## Supported Validation Domains

The validator supports validation across multiple CCSTMS domains defined in the ERA extended ontology:

| Domain | Description |
|--------|-------------|
| **Infrastructure** | Railway infrastructure topology, nodes, links, and all its related assets |
| **Engineering** | Engineering specifications and technical parameters |
| **Operational Plan** | Train scheduling, routing, and operational planning data |
| **Train Protection** | ETCS and train protection system configurations |
| **Localization** | Location referencing and positioning data |
| **On-board Infrastructure** | ATO (Automatic Train Operation) and on-board system specifications |
| **Subset026** | Subset 026 messaging and protocol validation |
| ***ETCS L2 Engineering** | ETCS Level2 engineering Rules (L2WoS) Conformance |

## Features

- Multi-domain validation against the Extended ERA ontology (CCSTMS)
- SHACL shapes for comprehensive constraint validation
- Multiple access channels: Web UI, REST API, SOAP API, and CLI
- Detailed validation reports with severity levels (Violation, Warning, Info)
- Support for multiple RDF serialization formats (Turtle, RDF/XML, JSON-LD, N-Triples)
- Integration with the ITB ecosystem for conformance testing

## Getting Started

### Using the Web Interface

Access the validator's web interface at:

```
https://www.itb.ec.europa.eu/shacl/CCSTMS/upload
```

Simply upload your RDF file, select the validation domain, and receive instant feedback on your data's conformance.

### Using the REST API

Validate RDF content programmatically via the REST API:

```bash
curl -X POST "https://www.itb.ec.europa.eu/shacl/CCSTMS/api/validate" \
  -H "Content-Type: application/json" \
  -d '{
    "contentToValidate": "<your-rdf-content>",
    "contentSyntax": "text/turtle",
    "validationType": "infrastructure"
  }'
```

For complete API documentation, see the [Swagger UI](https://www.itb.ec.europa.eu/shacl/swagger-ui.html).

### Using the SOAP API

For enterprise integration, the SOAP API is available at:

```
https://www.itb.ec.europa.eu/shacl/soap/CCSTMS/validation?wsdl
```

### Using the Command-Line Tool (CLI)

Download the offline validator for local or CI/CD pipeline usage:

```bash
# Download the CLI tool
wget https://www.itb.ec.europa.eu/shacl-offline/CCSTMS/validator.zip
unzip validator.zip

# Dependency
Ensure you have Java running on your workstation (minimum version 17).

# Run validation
java -jar validator.jar 
```

## Configuration

The validator is configured via `config.properties` in accordance with the ITB documentation/ guideline:

```properties
# Validation types (domains)
validator.type = infra, eng, ...

# Labels to describe the defined types
validator.typeLabel.infra = Infrastructure
validator.typeLabel.eng = Engineering 
# ... additional labels



# Validation artefacts mappings
validator.shaclFile.infra.v1.3 = shapes/infra-shapes_v1.3.ttl, shapes/infra-dataprep-shapes_v1.3.ttl
validator.shaclFile.eng.v1.3 = shapes/infra-shapes_v1.3.ttl, shapes/infra-dataprep-shapes_v1.3.ttl, shapes/eng-shapes_v1.3.ttl, ...
# ... additional mappings
```

For complete configuration options, refer to the [ITB RDF Validation Guide](https://www.itb.ec.europa.eu/docs/guides/latest/validatingRDF/).


### Validation Severity Levels

The validator uses three severity levels as defined by SHACL:

- **Violation** (`sh:Violation`): Critical errors that must be fixed
- **Warning** (`sh:Warning`): Recommendations that should be addressed
- **Info** (`sh:Info`): Informational messages for best practices

## Related Resources

- [ERA Ontology Repository](https://gitlab.com/era-europa-eu/public/interoperable-data-programme/era-ontology/era-ontology/-/tree/ext-ccstms)
- [ERA Website](https://www.era.europa.eu/)
- [Europe's Rail Joint Undertaking](https://rail-research.europa.eu/system_pillar/)
- [ITB RDF Validation Guide](https://www.itb.ec.europa.eu/docs/guides/latest/validatingRDF/)
- [SHACL Specification](https://www.w3.org/TR/shacl/)

## Contributing

Contributions to improve the SHACL shapes or extend validation coverage are welcome. Please follow these steps:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/improved-shapes`)
3. Commit your changes (`git commit -am 'Add new validation rules for X'`)
4. Push to the branch (`git push origin feature/improved-shapes`)
5. Open a Pull Request

When contributing SHACL shapes, please ensure:

- Shapes are well-documented with comments
- Messages are provided in English (and other languages if applicable), and mention the applicable CCSTMS version
- Test cases are provided
- Shapes validate correctly using the [ITB SHACL Shape Validator](https://www.itb.ec.europa.eu/shacl/shacl/upload)

## License

This project is licensed under the European Union Public License (EUPL) v1.2. See the [LICENSE](https://eupl.eu/1.2/en/) file for details.

## Support

For questions, issues, or feature requests:

- **Technical Support**: [DIGIT-ITB@ec.europa.eu](mailto:DIGIT-ITB@ec.europa.eu) | [f.ajala@lawconcon.com](mailto:f.ajala@lawconcon.com)
- **Issue Tracker**: [GitHub Issues](https://github.com/ISAITB/validator-resources-era-ccs-tms/issues)
- **ITB Documentation**: [https://www.itb.ec.europa.eu/docs/](https://interoperable-europe.ec.europa.eu/collection/interoperability-test-bed-repository/solution/interoperability-test-bed/documentation)

---

*Developed as part of the European railway interoperability programme*