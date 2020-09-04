# Publishing Infrastructure

## Grant RBAC Permissions
```
oc apply -f https://raw.githubusercontent.com/crossplane/crossplane/release-0.12/docs/snippets/publish/clusterrole.yaml
```

## Create an InfrastructureDefinition
This declares the schema for a PostgreSQLInstance
```
oc apply -f deploy/definitions_compositions/infrastructure_definition.yaml
```

## Create an InfrastructurePublication
```
oc apply -f deploy/definitions_compositions/infrastructure_definition.yaml
```

## Create a Composition
```
oc apply -f deploy/definitions_compositions/infrastructure_compositions.yaml
```

## Create a Requirement
```
oc apply -f deploy/definitions_compositions/infrastructure_requirement.yaml
```
