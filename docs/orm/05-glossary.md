# 05 Glossary

## Reading guide

- Audience: Java developers extending Openbravo modules who read [01-architecture.md](./01-architecture.md), [02-runtime-model.md](./02-runtime-model.md), [03-dal-service-api.md](./03-dal-service-api.md) and [04-security-and-filtering.md](./04-security-and-filtering.md) and need the exact meaning of a Data Access Layer (DAL), Hibernate or Openbravo term used there.
- Covers: every domain term that files 01 to 04 link to this glossary. Each first use of a term in those files links to its entry here.
- Entry layout: each entry under [Terms](#terms) has a level-3 heading with the term, one definition paragraph whose sentences each end with a `Class#method` or `build.xml#target` citation, and a `Used in:` line that links to the section of each file where the term is first used. The citations repeat the ones the owning file gives for the same behaviour, and the owning file holds the full treatment. Other entries of this glossary are linked by name inside a definition.
- Domain-context entries: [Application Dictionary (AD)](#application-dictionary-ad), [Business object](#business-object), [Client](#client), [Organization](#organization) and [Runtime model](#runtime-model) also carry a `Code qualification:` bullet that states, with citations, how the code narrows the general meaning.
- Conventions: classes outside the documented scope, such as `DynamicOBObject`, `DalThreadHandler`, `SessionInfo`, `DalUUIDGenerator`, `CriteriaImpl`, the AD model classes and the generated classes, are named only, and nothing is stated about their internals. Tests are not used as evidence in this file. Line numbers are not used.
- Reading order: 01 -> 02 -> 03 -> 04 -> 05. Previous file: [04-security-and-filtering.md](./04-security-and-filtering.md). This file is the last of the set, so there is no next file.

Sources: every class cited in this file resolves to the path below. Build targets are cited from the root `build.xml` and from `src/build.xml`.

| Class | Repository path |
| --- | --- |
| `BaseOBObject` | `src/org/openbravo/base/structure/BaseOBObject.java` |
| `DalMappingGenerator` | `src/org/openbravo/dal/core/DalMappingGenerator.java` |
| `DalRequestFilter` | `src/org/openbravo/dal/core/DalRequestFilter.java` |
| `DalSessionFactory` | `src/org/openbravo/dal/core/DalSessionFactory.java` |
| `DalSessionFactoryController` | `src/org/openbravo/dal/core/DalSessionFactoryController.java` |
| `DalUtil` | `src/org/openbravo/dal/core/DalUtil.java` |
| `Entity` | `src/org/openbravo/base/model/Entity.java` |
| `EntityAccessChecker` | `src/org/openbravo/dal/security/EntityAccessChecker.java` |
| `GenerateEntitiesTask` | `src/org/openbravo/base/gen/GenerateEntitiesTask.java` |
| `Identifiable` | `src/org/openbravo/base/structure/Identifiable.java` |
| `ModelProvider` | `src/org/openbravo/base/model/ModelProvider.java` |
| `ModelSessionFactoryController` | `src/org/openbravo/base/model/ModelSessionFactoryController.java` |
| `OBContext` | `src/org/openbravo/dal/core/OBContext.java` |
| `OBCriteria` | `src/org/openbravo/dal/service/OBCriteria.java` |
| `OBDal` | `src/org/openbravo/dal/service/OBDal.java` |
| `OBInterceptor` | `src/org/openbravo/dal/core/OBInterceptor.java` |
| `OBProvider` | `src/org/openbravo/base/provider/OBProvider.java` |
| `OBQuery` | `src/org/openbravo/dal/service/OBQuery.java` |
| `OrganizationStructureProvider` | `src/org/openbravo/dal/security/OrganizationStructureProvider.java` |
| `Property` | `src/org/openbravo/base/model/Property.java` |
| `SecurityChecker` | `src/org/openbravo/dal/security/SecurityChecker.java` |
| `SessionHandler` | `src/org/openbravo/dal/core/SessionHandler.java` |

`Entity` always means the runtime-model class above, never the annotation in `src/org/openbravo/base/Entity.java`.

## Terms

### Access level

The access level is the code of an AD table that `Entity#initialize(Table)` maps to an `AccessLevel` value and an `AccessLevelChecker` (boundary) of the same name: `1` is `ORGANIZATION`, `3` is `CLIENT_ORGANIZATION`, `4` is `SYSTEM`, `6` is `SYSTEM_CLIENT` and `7` is `ALL`, and any other code fails with `Check.fail` (`Entity#initialize(Table)`). `Entity#checkAccessLevel(String, String)` passes the entity name and the ids of a [client](#client) and an [organization](#organization) to that checker, and its Javadoc states that it throws `OBSecurityException` when they are not valid for the access level (`Entity#checkAccessLevel`). `SecurityChecker#checkWriteAccess` calls it as its last statement in every mode, [admin mode](#admin-mode) included, and its source comment says that the access-level check must also be done for administrators (`SecurityChecker#checkWriteAccess`).

Used in: [02-runtime-model](./02-runtime-model.md#access-level), [04-security-and-filtering](./04-security-and-filtering.md#reading-guide)

### Active filter

The active filter is the Hibernate session filter `activeFilter`, which `DalMappingGenerator#generateMapping(Entity)` declares with the condition `:activeParam = isActive` in the mapping of every active-enabled entity, and which `DalMappingGenerator#generateOneToMany` attaches to every [bag](#bag) whose target entity is active-enabled (`DalMappingGenerator#generateMapping(Entity)`, `DalMappingGenerator#generateOneToMany`). `OBDal#enableActiveFilter()` enables it on the [session](#session) of the instance's [pool](#pool) with `activeParam` set to `Y`, `OBDal#disableActiveFilter()` disables it, and `OBDal#isActiveFilterEnabled()` reports whether it is enabled (`OBDal#enableActiveFilter()`). It is separate from the active restriction that `OBCriteria` and `OBQuery` add under their own `setFilterOnActive(boolean)` switch, which defaults to true and does not read the session filter (`OBCriteria#initialize`, `OBQuery#addOrgClientActiveFilter`). The Javadoc of `OBDal#enableActiveFilter()` says the method overrides those switches, and [04-security-and-filtering.md, The session active filter](./04-security-and-filtering.md#the-session-active-filter) records this as an ambiguity that it does not resolve (`OBDal#enableActiveFilter()`).

Used in: [01-architecture](./01-architecture.md#per-entity-rules), [03-dal-service-api](./03-dal-service-api.md#obdalenableactivefilter), [04-security-and-filtering](./04-security-and-filtering.md#reading-guide)

### Admin mode

Admin mode is the state in which `OBContext#isInAdministratorMode()` is true: the top of the thread's [admin mode stack](#admin-mode-stack) is an admin entry pushed by `OBContext#setAdminMode(boolean)`, the deprecated flag of `OBContext#setInAdministratorMode(boolean)` is set, or the [role](#role) is `"0"` (`OBContext#isInAdministratorMode()`). In admin mode the [entity access](#entity-access) read and write checks are skipped, and `OBDal#save(Object)` calls neither `EntityAccessChecker#checkWritable(Entity)` nor `SecurityChecker#checkWriteAccess`, while the [access level](#access-level) check still runs (`OBDal#save(Object)`, `SecurityChecker#checkWriteAccess`). The client, organization and active restrictions of `OBCriteria` and `OBQuery` are added in admin mode too, because only their read check depends on `OBContext#isInAdministratorMode()` (`OBCriteria#initialize`). Module code calls `OBContext#setAdminMode(boolean)` immediately before a `try` block and `OBContext#restorePreviousMode()` in its `finally` block, so that each push is matched by a pop (`OBContext#setAdminMode(boolean)`, `OBContext#restorePreviousMode()`). `OBContext#setAdminMode()` calls `OBContext#setAdminMode(boolean)` with `false`, which switches the [org/client access check](#orgclient-access-check) off, although its Javadoc says entity access will also be checked; 04 records this as an ambiguity (`OBContext#setAdminMode()`). The matrix of what each mode skips is in [04-security-and-filtering.md, What admin mode skips](./04-security-and-filtering.md#what-admin-mode-skips) (`OBContext#doOrgClientAccessCheck()`).

Used in: [01-architecture](./01-architecture.md#layers), [02-runtime-model](./02-runtime-model.md#values-identity-and-the-new-object-flag), [03-dal-service-api](./03-dal-service-api.md#obdalsaveobject), [04-security-and-filtering](./04-security-and-filtering.md#reading-guide)

### Admin mode stack

The admin mode stack is the per-thread stack of `OBAdminMode` (boundary) entries onto which `OBContext#setAdminMode(boolean)` pushes an entry marked as admin mode and carrying the passed org/client flag, and from which `OBContext#restorePreviousMode()` pops, logging an unbalanced-call warning when the stack is empty (`OBContext#setAdminMode(boolean)`, `OBContext#restorePreviousMode()`). A separate stack holds the entries of [cross-org reference admin mode](#cross-org-reference-admin-mode), which `OBContext#setCrossOrgReferenceAdminMode()` pushes and `OBContext#restorePreviousCrossOrgReferenceMode()` pops (`OBContext#setCrossOrgReferenceAdminMode()`). When the admin stack is empty after a pop and the thread holds the shared admin context, `OBContext#restorePreviousMode()` removes that context from the thread (`OBContext#restorePreviousMode()`). `OBContext#clearAdminModeStack()` logs the unbalanced-call warning for each non-empty stack and clears both, and `DalRequestFilter#doFilter` calls it at the end of each request (`OBContext#clearAdminModeStack()`, `DalRequestFilter#doFilter`). `OBContext#initialize(String, String, String, String, String, String)` pushes a temporary admin entry directly onto the stack while it resolves a context, and pops it in a `finally` block (`OBContext#initialize(String, String, String, String, String, String)`).

Used in: [01-architecture](./01-architecture.md#request-lifecycle), [04-security-and-filtering](./04-security-and-filtering.md#initialization-order)

### Application Dictionary (AD)

The Application Dictionary is the set of AD records from which `ModelProvider#initialize` builds the [runtime model](#runtime-model): through the [model session factory](#model-session-factory) it reads all `Table` records sorted by name, all `Reference` records, all `Column` records, the `RefTable`, `RefSearch` and `RefList` records and the active `Module` records, and it creates one [entity](#entity) per remaining table (`ModelProvider#initialize`). `ModelProvider#initialize` applies no module filter, so a table that a module adds to the AD reaches the runtime model by the same path as a core table (`ModelProvider#initialize`, `ModelProvider#removeInvalidTables`). The AD model classes are boundary names, and this documentation does not describe AD tables, their columns or the database schema (`ModelSessionFactoryController#mapModel`).

- Code qualification: besides tables and columns, the model session factory maps the AD classes for references, reference lists, reference searches, reference tables, modules and packages (`ModelSessionFactoryController#mapModel`). `ModelProvider#initializeReferenceClasses` also registers with it the classes that reference [domain types](#domain-type) supply through their `getClasses()` (`ModelProvider#initializeReferenceClasses`). Unique constraints and column not-null flags come from database metadata, not from the AD (`ModelProvider#buildUniqueConstraints`, `ModelProvider#getColumnMandatories`). Some runtime properties, the one-to-many [child properties](#child-property) on parent entities, come from no AD column: `ModelProvider#createChildProperty` creates them (`ModelProvider#createChildProperty`).

Used in: [01-architecture](./01-architecture.md#reading-guide), [02-runtime-model](./02-runtime-model.md#reading-guide)

### Audit properties

The audit properties are the four properties `creationDate`, `createdBy`, `updated` and `updatedBy`: `Property#initializeName` marks a primitive `Date` property named `creationDate` or `updated`, and a reference property named `createdBy` or `updatedBy`, as audit info, comparing case-insensitively and normalizing the casing (`Property#initializeName`). `Entity#isTraceable` is true when exactly four properties of an entity are audit properties, and such an entity's generated class implements `Traceable` (`Entity#isTraceable`, `Entity#getImplementsStatement`). For a `Traceable` object, `OBInterceptor#doEvent` calls `OBInterceptor#onNew` when any audit property is null and `OBInterceptor#onUpdate` otherwise (`OBInterceptor#doEvent`). `OBInterceptor#onNew` sets each null audit property, the dates to the current date and the users to a [proxy](#proxy) of the context user, and `OBInterceptor#onUpdate` sets the updated date and the updated-by user (`OBInterceptor#onNew`, `OBInterceptor#onUpdate`). While `OBInterceptor#setPreventUpdateInfoChange(boolean)` is true in admin mode, `OBInterceptor#onUpdate` leaves `updated` and `updatedBy` unchanged (`OBInterceptor#onUpdate`).

Used in: [01-architecture](./01-architecture.md#request-lifecycle), [02-runtime-model](./02-runtime-model.md#flags), [04-security-and-filtering](./04-security-and-filtering.md#interceptor-behavior)

### Bag

A bag is the Hibernate collection mapping that `DalMappingGenerator#generateOneToMany` emits for every [one-to-many property](#one-to-many-property): an inverse bag (`inverse="true"`) keyed on the column of the referenced property, with `not-null="true"` on the key when that property is mandatory, and ordered by the order-by properties of the target entity (`DalMappingGenerator#generateOneToMany`). The bag of a [child property](#child-property) gets `cascade="all,delete-orphan"`, a bag whose owning or target entity is a [view entity](#view-entity) is `mutable="false"` and has no [cascade](#cascade), and a bag whose target entity is active-enabled carries the [active filter](#active-filter) (`DalMappingGenerator#generateOneToMany`).

Used in: [01-architecture](./01-architecture.md#per-entity-rules)

### Business object

A business object is an instance of a subclass of `BaseOBObject`, the abstract class that its Javadoc calls the root of the inheritance tree of all business objects; a subclass implements the abstract `BaseOBObject#getEntityName`, and `BaseOBObject#getEntity` resolves the object's [entity](#entity) once through `ModelProvider#getEntity(String)` (`BaseOBObject#getEntityName`, `BaseOBObject#getEntity`). Its values are kept in one array indexed by `Property#getIndexInEntity`, and code reads and writes them through the [dynamic API](#dynamic-api) or the [typed API](#typed-api) of its [generated class](#generated-class) (`BaseOBObject#getValue`, `BaseOBObject#get(String)`). `BaseOBObject` implements `BaseOBObjectDef`, `Identifiable`, `DynamicEnabled`, `OBNotSingleton` and `java.io.Serializable`, and `OBDal` reads, saves and removes business objects in the session of a [pool](#pool) (`BaseOBObject#get(String)`, `OBDal#save(Object)`).

- Code qualification: when the class named by `Entity#getClassName()` cannot be loaded, `Entity#getMappingClass()` returns null, and its Javadoc states that the system then uses a [DynamicOBObject](#dynamicobobject) as the runtime class (`Entity#getMappingClass()`). The code establishes nothing further about `DynamicOBObject`, which is a boundary class, so no type hierarchy is stated for it (`Entity#getMappingClass()`).

Used in: [01-architecture](./01-architecture.md#layers), [02-runtime-model](./02-runtime-model.md#baseobobject-and-its-interfaces), [03-dal-service-api](./03-dal-service-api.md#reading-guide), [04-security-and-filtering](./04-security-and-filtering.md#cross-organization-reference-check)

### Cascade

Cascade is the Hibernate mapping attribute that the generated mapping sets on parent references and on bags: `DalMappingGenerator#generateReferenceMapping` adds `cascade="persist"` to every mandatory parent reference, and `DalMappingGenerator#generateOneToMany` adds `cascade="all,delete-orphan"` to the [bag](#bag) of a [child property](#child-property) and omits cascade when the owning or the target entity is a [view entity](#view-entity) (`DalMappingGenerator#generateReferenceMapping`, `DalMappingGenerator#generateOneToMany`). The source comment in `DalMappingGenerator#generateReferenceMapping` gives the reason for `persist`: it prevents cascade errors in which the parent would be saved after the child (`DalMappingGenerator#generateReferenceMapping`).

Used in: [01-architecture](./01-architecture.md#per-entity-rules)

### Child property

A child property is a [one-to-many property](#one-to-many-property) that `ModelProvider#createChildProperty` adds to a parent entity for a property of a child entity that references it: the new property uses `OneToManyDomainType`, targets the child entity, references the child property and is not mandatory (`ModelProvider#createChildProperty`). It comes from no AD column, and `ModelProvider#createPropertyInParentEntity` creates one for each property of an entity that is neither a [datasource-based entity](#datasource-based-entity) nor an [HQL-based entity](#hql-based-entity) and that `ModelProvider#shouldGenerateChildPropertyInParent` accepts (`ModelProvider#createPropertyInParentEntity`, `ModelProvider#initialize`). That method accepts a property flagged as child property in parent, or any eligible property when the `hb.generate.all.parent.child.properties` setting is true (`ModelProvider#shouldGenerateChildPropertyInParent`). The new property is also marked as a child, the flag that `Property#isChild` reads, only when the child property is a parent link, and `DalMappingGenerator#generateOneToMany` gives a one-to-many property marked as a child `cascade="all,delete-orphan"` (`ModelProvider#createChildProperty`, `DalMappingGenerator#generateOneToMany`).

Used in: [01-architecture](./01-architecture.md#per-entity-rules), [02-runtime-model](./02-runtime-model.md#how-the-runtime-model-is-built)

### Client

A client is the record that a business object of a client-enabled entity references through its property named `client`; `Property#initializeName` sets the entity's client-enabled flag when it finds that property (`Property#initializeName`). `OBContext#initialize(String, String, String, String, String, String)` resolves the current client of the [user context](#user-context), and `OBContext#setReadableClients(Role)` computes the [readable clients](#readable-clients) from the [role](#role) (`OBContext#initialize(String, String, String, String, String, String)`, `OBContext#setReadableClients(Role)`). When the context is outside admin mode or the [org/client access check](#orgclient-access-check) is on, `SecurityChecker#checkWriteAccess` requires the client of a client-enabled object to equal `OBContext#getCurrentClient()`, and `OBDal#save(Object)` fills a missing client with a [proxy](#proxy) of the current client (`SecurityChecker#checkWriteAccess`, `OBDal#setClientOrganization`).

- Code qualification: filtering by client is applied by `OBCriteria#initialize` and `OBQuery#addOrgClientActiveFilter`, and only for client-enabled entities (`OBCriteria#initialize`, `OBQuery#addOrgClientActiveFilter`). The filter can be switched off per query with `OBCriteria#setFilterOnReadableClients(boolean)` or `OBQuery#setFilterOnReadableClients(boolean)` (`OBCriteria#setFilterOnReadableClients(boolean)`, `OBQuery#setFilterOnReadableClients(boolean)`). It is added in admin mode too, because only the read check of these methods depends on `OBContext#isInAdministratorMode()` (`OBCriteria#initialize`). `OBDal#get(Class, Object)` and `OBDal#get(String, Object)` add no row filter and perform only the entity-level read check (`OBDal#get(Class, Object)`, `OBDal#get(String, Object)`).

Used in: [01-architecture](./01-architecture.md#per-entity-rules), [02-runtime-model](./02-runtime-model.md#access-level), [03-dal-service-api](./03-dal-service-api.md#reading-guide), [04-security-and-filtering](./04-security-and-filtering.md#reading-guide)

### Composite id

A composite id is the id of an entity that has several id properties: `ModelProvider#createCompositeId` creates a property named `id` whose id parts are the former id properties, each marked as part of the composite id (`ModelProvider#createCompositeId`). Its type name is the entity class name plus `.Id`, and its formatted default value is `new Id()` (`Property#getTypeName`, `Property#getFormattedDefaultValue`). `DalMappingGenerator#generateCompositeID` maps it as `<composite-id name="id">` with the class `<ClassName>$Id`, a `<key-property>` for each primitive part and a `<key-many-to-one>` for each reference part, and `DalMappingGenerator#generateMapping(Entity)` does not emit the parts as ordinary properties (`DalMappingGenerator#generateCompositeID`, `DalMappingGenerator#generateMapping(Entity)`).

Used in: [01-architecture](./01-architecture.md#which-entities-are-mapped), [02-runtime-model](./02-runtime-model.md#how-the-runtime-model-is-built)

### Computed column

A computed column is a property with SQL logic, for which `Property#isComputedColumn` is true; `Entity#hasComputedColumns` is true when at least one property of the entity is one (`Property#isComputedColumn`, `Entity#hasComputedColumns`). For such an entity `ModelProvider#initialize` adds a [virtual entity](#virtual-entity) whose name ends in `_ComputedColumns`, and `Entity#initialize(Table)` adds the [proxy](#proxy) property `_computedColumns` (`ModelProvider#initialize`, `Entity#initialize(Table)`). `DalMappingGenerator#generateComputedColumnsMapping` maps that property as a read-only [many-to-one](#many-to-one) on the id column, and `DalMappingGenerator#generateComputedColumnsClassMapping` appends the companion class mapping that holds the SQL-logic properties (`DalMappingGenerator#generateComputedColumnsMapping`, `DalMappingGenerator#generateComputedColumnsClassMapping`). `GenerateEntitiesTask#execute` writes a `<SimpleClassName>_ComputedColumns.java` class for an entity with computed columns (`GenerateEntitiesTask#execute`). `Entity#getRealProperties(boolean)` leaves computed columns out unless its argument is `true`, while its Javadoc describes the flag the other way round; 02 records this as an ambiguity (`Entity#getRealProperties`).

Used in: [01-architecture](./01-architecture.md#per-entity-rules), [02-runtime-model](./02-runtime-model.md#how-the-runtime-model-is-built)

### Criteria

A criteria is a Hibernate criteria query object: `OBCriteria` extends Hibernate's `CriteriaImpl` (boundary), and `OBDal#createCriteria(Class)` and its overloads run the read check and return an `OBCriteria` bound to the session of the instance's [pool](#pool) (`OBDal#createCriteria(Class)`). `OBCriteria#list()`, `OBCriteria#count()`, both `scroll` overloads and `OBCriteria#uniqueResult()` first call `OBCriteria#initialize`, which adds the read check outside admin mode and the organization, client and active restrictions, and then delegate to the superclass (`OBCriteria#list()`, `OBCriteria#initialize`). When `OBCriteria#initialize` runs again after the criteria was modified, it logs a warning that filters may be duplicated (`OBCriteria#initialize`).

Used in: [03-dal-service-api](./03-dal-service-api.md#reading-guide), [04-security-and-filtering](./04-security-and-filtering.md#criteria-restrictions)

### Cross-org reference admin mode

Cross-org reference admin mode is the mode that `OBContext#setCrossOrgReferenceAdminMode()` enters by pushing, onto a stack separate from the [admin mode stack](#admin-mode-stack), an entry that is not admin mode, keeps the org/client check and is marked as cross-org admin mode (`OBContext#setCrossOrgReferenceAdminMode()`). Its Javadoc states that the mode allows references from an object to another one outside the [natural tree](#natural-tree) of its organization, only for columns marked to allow it (`OBContext#setCrossOrgReferenceAdminMode()`). `OBContext#isInCrossOrgAdministratorMode()` is true while the top of that stack is such an entry, and `OBContext#restorePreviousCrossOrgReferenceMode()` pops it (`OBContext#isInCrossOrgAdministratorMode()`, `OBContext#restorePreviousCrossOrgReferenceMode()`). While the mode is on, the [cross-organization reference check](#cross-organization-reference-check) skips the properties that allow cross-organization references (`OBInterceptor#checkReferencedOrganizations`).

Used in: [04-security-and-filtering](./04-security-and-filtering.md#the-admin-mode-stack)

### Cross-organization reference check

The cross-organization reference check is `OBInterceptor#checkReferencedOrganizations`, which, for an organization-enabled object, throws `OBSecurityException` when a reference whose value is an organization-enabled business object or proxy, and not an `Organization`, points to an organization outside the [natural tree](#natural-tree) of the object's organization, as reported by `OrganizationStructureProvider#isInNaturalTree(Organization, Organization)` (`OBInterceptor#checkReferencedOrganizations`). It examines the changed references, or every reference when the object is new or its organization changed, and skips references to `AttributeSetInstance` (boundary) and to [virtual entities](#virtual-entity), audit-info and image properties, and properties that allow cross-organization references while [cross-org reference admin mode](#cross-org-reference-admin-mode) is on (`OBInterceptor#checkReferencedOrganizations`). `OBInterceptor#onFlushDirty` runs it unless `OBInterceptor#setDisableCheckReferencedOrganizations(boolean)` has set the thread-local switch to true (`OBInterceptor#onFlushDirty`, `OBInterceptor#setDisableCheckReferencedOrganizations`). The call for new records in `OBInterceptor#onSave` is commented out, so the check runs only from `OBInterceptor#onFlushDirty`, although the method still computes an `isNew` flag; 04 records this as an ambiguity (`OBInterceptor#onSave`, `OBInterceptor#checkReferencedOrganizations`).

Used in: [04-security-and-filtering](./04-security-and-filtering.md#interceptor-behavior)

### Datasource-based entity

A datasource-based entity is an entity whose AD table has the data origin `Datasource`; `Entity#initialize(Table)` sets its datasource-based flag, read by `Entity#isDataSourceBased()`, from that value (`Entity#initialize(Table)`). `GenerateEntitiesTask#execute` writes no class for it, `DalMappingGenerator#generateMapping()` does not map it, and `ModelProvider#initialize` creates no [child properties](#child-property) for it and does not take its mandatory flags from the database (`GenerateEntitiesTask#execute`, `DalMappingGenerator#generateMapping()`, `ModelProvider#initialize`).

Used in: [01-architecture](./01-architecture.md#which-entities-are-mapped), [02-runtime-model](./02-runtime-model.md#how-the-runtime-model-is-built)

### Deactivated organization

A deactivated organization is an organization of an active role-organization assignment of the current [role](#role) whose organization record is inactive, or, for an automatic role, an organization that `RoleAccessUtils` (boundary) returns; `OBContext#getDeactivatedOrganizations()` computes the set on first use, caches it and returns a copy (`OBContext#getDeactivatedOrganizations()`). `SecurityChecker#checkWriteAccess` lets an `Organization` object that is among the deactivated organizations pass the [writable organizations](#writable-organizations) test (`SecurityChecker#checkWriteAccess`). A deactivated organization can still be among the [readable organizations](#readable-organizations), because the `"0"` branch of `OBContext#setReadableOrganizations(Role)` selects the organizations of the client with no condition on the active flag (`OBContext#getOrganizations(Client)`).

Used in: [04-security-and-filtering](./04-security-and-filtering.md#reading-guide)

### Derived readable

An entity is derived readable for a role when the `EntityAccessChecker` of the context classifies it so: per the Javadoc of `EntityAccessChecker#initialize`, the checker computes the readable, writable, non-readable and derived-readable entities from the windows available to the role (`EntityAccessChecker#initialize`). `EntityAccessChecker#isDerivedReadable(Entity)` reports the classification and returns false before the checker is initialized and in admin mode (`EntityAccessChecker#isDerivedReadable`). For an object of a derived-readable entity, `BaseOBObject#checkDerivedReadable` throws `OBSecurityException` when a property is read or set whose `Property#allowDerivedRead` is false, unless the object is marked allow-read through `BaseOBObject#setAllowRead(boolean)`, no initialized `OBContext` exists, or the context is in admin mode (`BaseOBObject#checkDerivedReadable`). `Property#allowDerivedRead` is true for the active column, [audit properties](#audit-properties), id and [identifier](#identifier) properties, client and organization properties, and properties that another property references (`Property#allowDerivedRead`). A non-generated child property is still flagged as being referenced, and the source comment in `ModelProvider#createPropertyInParentEntity` gives the reason: the flag affects `BaseOBObject#checkDerivedReadable` (`ModelProvider#createPropertyInParentEntity`). `EntityAccessChecker#checkReadable(Entity)` accepts an entity that is readable or derived readable (`EntityAccessChecker#checkReadable`).

Used in: [02-runtime-model](./02-runtime-model.md#derived-read), [04-security-and-filtering](./04-security-and-filtering.md#access-checks)

### Development time

Development time is the build phase in which `GenerateEntitiesTask#main`, started by `src/build.xml#generate.entities.quick`, writes the [generated classes](#generated-class), as opposed to the runtime in which the remaining DAL classes run inside the application (`GenerateEntitiesTask#main`). At development time `GenerateEntitiesTask#execute` reads the same [runtime model](#runtime-model) that the running application uses, through `ModelProvider#getModel` (`GenerateEntitiesTask#execute`).

Used in: [01-architecture](./01-architecture.md#reading-guide)

### Dirty check

A dirty check asks whether a Hibernate [session](#session) holds unflushed changes: `SessionHandler#isSessionDirty(String)` asks Hibernate for the pool's session while a thread-local flag is set, and `OBInterceptor#onFlushDirty` returns false at once while `SessionHandler#isCheckingDirtySession()` reports that flag (`SessionHandler#isSessionDirty(String)`, `OBInterceptor#onFlushDirty`). The Javadoc of `SessionHandler#isSessionDirty(String)` gives the reason: calling Hibernate's `Session#isDirty()` directly triggers the entity persistence observers of modified entities, and this method handles the check so that they are not called (`SessionHandler#isSessionDirty(String)`). `OBDal#isSessionDirty()` returns the result for the instance's [pool](#pool), and `SessionHandler#flushRemainingChanges` flushes again while it reports changes (`OBDal#isSessionDirty()`, `SessionHandler#flushRemainingChanges`).

Used in: [01-architecture](./01-architecture.md#dirty-check), [03-dal-service-api](./03-dal-service-api.md#obdalissessiondirty), [04-security-and-filtering](./04-security-and-filtering.md#callbacks)

### Domain type

A domain type is the class behind an AD reference that the runtime model uses for the type of a property: `ModelProvider#initialize` sets the model provider on the domain type of each reference and initializes it, and `Property#initializeFromColumn` copies the domain type from the column (`ModelProvider#initialize`, `Property#initializeFromColumn`). `Property#isPrimitive` is true when the domain type is a `PrimitiveDomainType`, `Property#checkIsValidValue` lets the domain type check a value, and the [child properties](#child-property) use `OneToManyDomainType` (`Property#isPrimitive`, `Property#checkIsValidValue`, `ModelProvider#createChildProperty`). `ModelProvider#initializeReferenceClasses` loads the reference implementation classes and, for each class assignable to `BaseDomainType` (boundary), registers the classes returned by its `getClasses()` with the [model session factory](#model-session-factory) (`ModelProvider#initializeReferenceClasses`). `src/build.xml#compile.src.gen` compiles the domain types of modules before generation runs (`src/build.xml#compile.src.gen`).

Used in: [01-architecture](./01-architecture.md#entity-generation-generateentities), [02-runtime-model](./02-runtime-model.md#how-the-runtime-model-is-built)

### Dynamic API

The dynamic API reads and writes the values of a [business object](#business-object) by property name through `BaseOBObject#get(String)` and `BaseOBObject#set(String, Object)`, the methods that `DynamicEnabled` and `BaseOBObjectDef` declare (`BaseOBObject#get(String)`, `BaseOBObject#set(String, Object)`). A read resolves the property with `Entity#getProperty(String)`, which throws `CheckException` for an unknown name, and runs `BaseOBObject#checkDerivedReadable` (`BaseOBObject#get(String, Language, String)`). A write runs `Property#checkIsValidValue`, `BaseOBObject#checkDerivedReadable` and `Property#checkIsWritable`, in that order, before it stores the value with `BaseOBObject#setValue` (`BaseOBObject#set`). `BaseOBObject#getValue(String)` and `BaseOBObject#setValue(String, Object)` give unchecked access by name, and `OBDal#get(String, Object)` loads an object by [entity name](#entity-name) for use with this API (`BaseOBObject#getValue`, `OBDal#get(String, Object)`). Writing a value of an inactive property through it throws `ValidationException` (`Property#checkIsWritable`).

Used in: [02-runtime-model](./02-runtime-model.md#reading-guide), [04-security-and-filtering](./04-security-and-filtering.md#access-checks)

### DynamicOBObject

`DynamicOBObject` is the class that, per the Javadoc of `Entity#getMappingClass`, the system uses as the runtime class of an entity whose generated class cannot be loaded, in which case `Entity#getMappingClass` returns null (`Entity#getMappingClass`). It is a boundary class, and this documentation states nothing about its type hierarchy or its members (`Entity#getMappingClass`). `DalMappingGenerator#generateMapping()` does not map an entity whose `Entity#getMappingClass()` returns null (`DalMappingGenerator#generateMapping()`).

Used in: [02-runtime-model](./02-runtime-model.md#mapping-class-and-generated-interfaces)

### Entity

An entity is the runtime-model object of class `Entity` that models one AD table; `Entity#initialize(Table)` sets its table name and table id, sets its class name to the table's package name, a dot and its class name, and sets its [entity name](#entity-name) from the name of the AD table record (`Entity#initialize(Table)`). It holds the [properties](#property) created from the table's columns and those added later, listed by `Entity#getProperties`, and the client-enabled, organization-enabled, active-enabled, view, datasource-based and HQL-based flags that the DAL tests (`Entity#getProperties`, `Entity#initialize(Table)`). `ModelProvider#getModel` returns the [runtime model](#runtime-model) as the list of entities, and `ModelProvider#getEntity(String)` looks one up by entity name, throwing `CheckException` for an unknown name (`ModelProvider#getModel`, `ModelProvider#getEntity(String)`). The kinds are table-based entities, [view entities](#view-entity), [datasource-based entities](#datasource-based-entity), [HQL-based entities](#hql-based-entity) and [virtual entities](#virtual-entity) (`Entity#initialize(Table)`, `Entity#initializeComputedColumns`).

Used in: [01-architecture](./01-architecture.md#reading-guide), [02-runtime-model](./02-runtime-model.md#reading-guide), [03-dal-service-api](./03-dal-service-api.md#reading-guide), [04-security-and-filtering](./04-security-and-filtering.md#reading-guide)

### Entity access

Entity access is the role-level decision whether an entity may be read or written, made by the `EntityAccessChecker` that `OBContext#getEntityAccessChecker` creates on first use through `OBProvider#get(Class)` and initializes with the role id and the context (`OBContext#getEntityAccessChecker`). `EntityAccessChecker#checkReadable(Entity)` throws `OBSecurityException` when the entity is non-readable for the role, or neither readable nor [derived readable](#derived-readable); outside admin mode it is called through the private `OBDal#checkReadAccess` from `OBDal#get(Class, Object)`, `OBDal#get(String, Object)` and every `OBDal#createQuery` and `OBDal#createCriteria` overload, and directly from `OBCriteria#initialize` and `OBQuery#createQueryString` (`EntityAccessChecker#checkReadable`). `EntityAccessChecker#checkWritable(Entity)` throws `OBSecurityException` when `EntityAccessChecker#isWritable(Entity)` is false, and `OBDal#save(Object)` calls it outside admin mode (`EntityAccessChecker#checkWritable`, `OBDal#save(Object)`). `OBDal#checkReadAccess` exempts the Client and Organization entities from the read check, while `OBCriteria#initialize` and `OBQuery#createQueryString` do not; 04 records this as an ambiguity (`OBDal#checkReadAccess`). `OBContext#setRole(Role)` discards the cached checker so that it is recomputed on next use (`OBContext#setRole(Role)`).

Used in: [04-security-and-filtering](./04-security-and-filtering.md#reading-guide)

### Entity name

The entity name is the name under which the runtime model, the [Hibernate mapping](#hibernate-mapping) and the DAL services identify an entity; `Entity#initialize(Table)` takes it from the name of the AD table record through `Entity#setName`, which drops illegal characters with `Entity#removeIllegalChars`, and `DalMappingGenerator#generateMapping(Entity)` writes it into the class mapping (`Entity#initialize(Table)`, `Entity#removeIllegalChars`, `DalMappingGenerator#generateMapping(Entity)`). `ModelProvider#getEntity(String)` looks entities up by it, and `SessionHandler#save(String, Object)` passes it to Hibernate's `saveOrUpdate` for an `Identifiable` object (`ModelProvider#getEntity(String)`, `SessionHandler#save(String, Object)`). A generated class exposes it in a public static `ENTITY_NAME` field, which `DalUtil#getEntityName(Class)` reads through reflection, and `DalUtil#getEntityName(Object)` resolves it for a Hibernate [proxy](#proxy) without loading the object (`DalUtil#getEntityName(Class)`, `DalUtil#getEntityName(Object)`). `OBInterceptor#getEntityName(Object)` returns it for a `BaseOBObject` and null for any other object (`OBInterceptor#getEntityName`).

Used in: [01-architecture](./01-architecture.md#layers), [02-runtime-model](./02-runtime-model.md#looking-up-the-model), [03-dal-service-api](./03-dal-service-api.md#obdalsaveobject), [04-security-and-filtering](./04-security-and-filtering.md#callbacks)

### Event handler

An event handler, in the source comment of `SessionHandler#flushRemainingChanges`, is a business event handler that can change data while a session is flushed (`SessionHandler#flushRemainingChanges`). For that reason `SessionHandler#flushRemainingChanges` calls `OBDal#flush()` again while `SessionHandler#isSessionDirty(String)` reports changes, so that the session is really clean before the commit, and after more than 100 flushes it logs "Infinite loop in flushing session, tried more than 100 flushes" and stops looping without throwing (`SessionHandler#flushRemainingChanges`).

Used in: [01-architecture](./01-architecture.md#commit-rollback-and-flush)

### Fetch size

The fetch size is the value that `OBQuery#setFetchSize(int)` stores and `OBQuery#getFetchSize()` returns; `OBQuery#createQuery(Class)` applies it to the Hibernate query it creates when the value is above `-1` (`OBQuery#setFetchSize(int)`, `OBQuery#createQuery(Class)`). `OBQuery#count()` does not apply it (`OBQuery#count()`).

Used in: [03-dal-service-api](./03-dal-service-api.md#obquerycount)

### Flush

A flush writes the pending changes of a Hibernate [session](#session) to the database: `OBDal#flush()` flushes the session of the instance's [pool](#pool) when the current thread's `SessionHandler` is present and available for it, and first passes the connection to `SessionInfo.saveContextInfoIntoDB` (boundary) when the session is dirty (`OBDal#flush()`). `SessionHandler#begin(String)` gives each session the [flush mode](#flush-mode) `FlushMode.COMMIT`, and `SessionHandler#commitAndClose(String)` calls `SessionHandler#flushRemainingChanges`, which flushes repeatedly while the session is dirty (`SessionHandler#begin(String)`, `SessionHandler#flushRemainingChanges`). After a flush, `OBInterceptor#postFlush` clears the [new object](#new-object) flag of every flushed object (`OBInterceptor#postFlush`). The `LocalInterceptor` of the [model session factory](#model-session-factory) fails on every dirty flush, which keeps the model session read-only (`ModelSessionFactoryController#setInterceptor`).

Used in: [01-architecture](./01-architecture.md#commit-rollback-and-flush), [02-runtime-model](./02-runtime-model.md#how-the-runtime-model-is-built), [03-dal-service-api](./03-dal-service-api.md#obdalcommitandclose), [04-security-and-filtering](./04-security-and-filtering.md#callbacks)

### Flush mode

The flush mode is the Hibernate session setting that decides when the session is flushed; `SessionHandler#begin(String)` sets it to `FlushMode.COMMIT` on every session it creates for a [pool](#pool), before it begins the [transaction](#transaction) (`SessionHandler#begin(String)`). `SessionHandler#commitAndClose(String)` calls `SessionHandler#flushRemainingChanges` before it commits (`SessionHandler#commitAndClose(String)`).

Used in: [01-architecture](./01-architecture.md#request-lifecycle)

### Foreign key

A foreign key is a column that references the primary key of another table; when the id property of an entity is also a reference, a foreign key, the source comment in `ModelProvider#initialize` says that Hibernate requires two mappings, one for the id and one for the reference, and that the id generation strategy should be `foreign` (`ModelProvider#initialize`). `ModelProvider#setVirtualPropertiesForReferenceId` handles such entities: for one non-primitive id property, `ModelProvider#createIdReferenceProperty` adds a [one-to-one](#one-to-one) [reference property](#reference-property) to the target entity and turns the original id into a [primitive property](#primitive-property) whose `Property#getIdBasedOnProperty` is the new property (`ModelProvider#setVirtualPropertiesForReferenceId`, `ModelProvider#createIdReferenceProperty`). `DalMappingGenerator#generateStandardID` maps an id based on another property with a `foreign` generator whose `property` parameter names that property (`DalMappingGenerator#generateStandardID`).

Used in: [02-runtime-model](./02-runtime-model.md#how-the-runtime-model-is-built)

### FreeMarker template

A FreeMarker template is a template file of the FreeMarker template engine, which the class Javadoc of `GenerateEntitiesTask` names as the engine used to generate the entities; `GenerateEntitiesTask#execute` loads `entity.ftl` and `entityComputedColumns.ftl` from `org/openbravo/base/gen` under the base path through `GenerateEntitiesTask#createTemplateImplementation` (`GenerateEntitiesTask#execute`). `GenerateEntitiesTask#processTemplate` fills them to write each [generated class](#generated-class) and its `_ComputedColumns` companion class (`GenerateEntitiesTask#processTemplate`). A failure to load or process a template is wrapped in `IllegalStateException` and propagates, and the templates themselves are named as inputs only (`GenerateEntitiesTask#createTemplateImplementation`, `GenerateEntitiesTask#processTemplate`).

Used in: [01-architecture](./01-architecture.md#entity-generation-generateentities)

### generate.entities

`generate.entities` is the Ant target that regenerates the [generated classes](#generated-class): `build.xml#generate.entities` delegates to `src/build.xml#generate.entities`, which has no body and depends on `src/build.xml#clean.src.gen` and `src/build.xml#generate.entities.quick` (`build.xml#generate.entities`). The quick target depends on `src/build.xml#compile.src.gen`, runs `GenerateEntitiesTask#main` with the source path, the `src-gen` path and `config/Openbravo.properties`, and then compiles the generated sources (`src/build.xml#generate.entities.quick`). `GenerateEntitiesTask#execute` skips generation unless `GenerateEntitiesTask#hasChanged` is true, which happens when `src-gen/org/openbravo/model/ad` does not exist or holds a file older than the later of the model update time and the newest modification time of the generator source packages (`GenerateEntitiesTask#hasChanged`). It runs directly with `ant generate.entities`, and the root targets `build.xml#compile.war` and `build.xml#db.apply.modules.sampledata` call it, while `build.xml#smartbuild` and `build.xml#update.database` call the quick target (`build.xml#compile.war`, `build.xml#smartbuild`).

Used in: [01-architecture](./01-architecture.md#reading-guide), [02-runtime-model](./02-runtime-model.md#looking-up-the-model)

### Generated class

A generated class is the Java class that `GenerateEntitiesTask#execute` writes under [src-gen](#src-gen) for each entity that is neither datasource-based, HQL-based nor virtual, at the path given by `Entity#getClassName()` with dots replaced by slashes (`GenerateEntitiesTask#execute`, `Entity#getClassName`). Its `implements` clause and imports come from `Entity#getImplementsStatement` and `Entity#getJavaImports`, and its accessor names from `Property#getGetterSetterName` (`Entity#getImplementsStatement`, `Entity#getJavaImports`, `Property#getGetterSetterName`). Per the comment of `BaseOBObject#setDefaultValue`, a constructor of the generated class sets default data through that method without a security check (`BaseOBObject#setDefaultValue`). At run time `Entity#getMappingClass` loads it, and returns null when it cannot be loaded (`Entity#getMappingClass`). The generated sources are not in the repository, so `SystemInformation` is recorded as Not Found in [02-runtime-model.md, Worked example: SystemInformation](./02-runtime-model.md#worked-example-systeminformation), and generated classes named in 01 to 04, such as `User`, `Role`, `Organization` and `Client`, are boundary names (`GenerateEntitiesTask#execute`).

Used in: [01-architecture](./01-architecture.md#layers), [02-runtime-model](./02-runtime-model.md#reading-guide), [03-dal-service-api](./03-dal-service-api.md#reading-guide), [04-security-and-filtering](./04-security-and-filtering.md#reading-guide)

### hbm file

An hbm file is a Hibernate XML mapping file: `DalSessionFactoryController#mapModel` writes the string from `DalMappingGenerator#generateMapping()` to a temporary file with the suffix `.hbm`, adds it to the Hibernate `Configuration` and deletes it in a `finally` block, or adds the file named by the `hibernate.hbm.file` setting directly when that setting is present (`DalSessionFactoryController#mapModel`). A failure to write the temporary file raises `OBException`, while a failure to delete it is only logged (`DalSessionFactoryController#mapModel`). `DalMappingGenerator#generateMapping()` reads the file named by `hibernate.hbm.file` instead of generating the mapping when the file exists and `hibernate.hbm.readFile` is `true`, and when the setting is present and the mapping is generated it writes the result to that file (`DalMappingGenerator#generateMapping()`). Its source comment says that reading the file is useful while developing changes in the mapping (`DalMappingGenerator#generateMapping()`).

Used in: [01-architecture](./01-architecture.md#registering-the-mapping)

### Hibernate mapping

The Hibernate mapping is the XML that maps the entities of the [runtime model](#runtime-model) to their tables: `DalMappingGenerator#generateMapping()` iterates `ModelProvider#getModel`, maps every entity except [datasource-based entities](#datasource-based-entity), [virtual entities](#virtual-entity) and entities whose `Entity#getMappingClass()` returns null, and inserts the class mappings into the `template_main.hbm.xml` template (`DalMappingGenerator#generateMapping()`). `DalMappingGenerator#generateMapping(Entity)` emits each class mapping, with its id, properties, references, [bags](#bag), [computed column](#computed-column) mapping and, for an active-enabled entity, the [active filter](#active-filter) definition (`DalMappingGenerator#generateMapping(Entity)`). `DalSessionFactoryController#mapModel` registers the result with Hibernate, directly or through an [hbm file](#hbm-file) (`DalSessionFactoryController#mapModel`).

Used in: [01-architecture](./01-architecture.md#reading-guide), [04-security-and-filtering](./04-security-and-filtering.md#the-session-active-filter)

### HQL

HQL is Hibernate's query language: `OBQuery` wraps an HQL where and order by clause, and `OBQuery#createQueryString` builds the full HQL from it, wrapping the caller's clause in parentheses after `where`, running the read check outside admin mode and appending the client, organization and active restrictions (`OBQuery#createQueryString`). The source comment in `OBQuery#createQueryString` gives the reason for the parentheses: the clauses it adds must all be and-ed with the caller's clause (`OBQuery#createQueryString`). `OBDal#createQuery(Class, String)` and its overloads take that clause, and `OBQuery#getWhereAndOrderBy()` replaces an upper-case `WHERE` keyword with `where` because, per its source comment, the upper-case keyword makes Hibernate's HQL parser throw an exception (`OBDal#createQuery(Class, String)`, `OBQuery#getWhereAndOrderBy()`). `OBQuery#createQuery(Class)` creates the Hibernate `Query` (boundary) from the built string (`OBQuery#createQuery(Class)`).

Used in: [03-dal-service-api](./03-dal-service-api.md#reading-guide), [04-security-and-filtering](./04-security-and-filtering.md#query-restrictions)

### HQL-based entity

An HQL-based entity is an entity whose AD table has the data origin `HQL`; `Entity#initialize(Table)` sets its HQL-based flag, read by `Entity#isHQLBased()`, from that value (`Entity#initialize(Table)`). `GenerateEntitiesTask#execute` writes no class for it, and `ModelProvider#initialize` creates no [child properties](#child-property) for it and does not take its mandatory flags from the database (`GenerateEntitiesTask#execute`, `ModelProvider#initialize`).

Used in: [01-architecture](./01-architecture.md#entity-generation-generateentities), [02-runtime-model](./02-runtime-model.md#how-the-runtime-model-is-built)

### HTTP session

The HTTP session is the servlet session of a request, in which the [user context](#user-context) is stored under the attribute `#OBContext`: `OBContext#setOBContext(HttpServletRequest)` returns without action when the request has no HTTP session, creates and initializes a context through `OBContext#setFromRequest(HttpServletRequest)` when none is stored, and re-initializes a stored context from the request when its user, role, client or organization differs from a value present in the HTTP session (`OBContext#setOBContext(HttpServletRequest)`). `OBContext#setFromRequest(HttpServletRequest)` reads the `#AD_User_ID`, `#AD_Role_ID`, `#AD_Client_ID` and `#AD_Org_ID` session values (`OBContext#setFromRequest(HttpServletRequest)`). `OBContext#setOBContextInSession(HttpServletRequest, OBContext)` writes the context to the attribute, refusing with a warning to store the admin context, and `DalRequestFilter#doFilter` calls it at the end of each request (`OBContext#setOBContextInSession(HttpServletRequest, OBContext)`, `DalRequestFilter#doFilter`). The `OBContext` class Javadoc states that the context can be serialized as part of the Tomcat persistent session mechanism, and `OBContext#readObject` re-runs initialization from the stored ids (`OBContext#readObject`).

Used in: [01-architecture](./01-architecture.md#request-lifecycle), [04-security-and-filtering](./04-security-and-filtering.md#creating-and-installing-a-context)

### Identifier

The identifier is the user-readable text of a [business object](#business-object): `Entity#initialize(Table)` collects the identifier properties of an entity from the columns flagged as identifier, sorted by sequence number, and `BaseOBObject#getIdentifier` returns the identifier that `IdentifierProvider` (boundary) computes for the object (`Entity#initialize(Table)`, `BaseOBObject#getIdentifier`). The Javadoc of `Identifiable` says that an identifiable object has a unique id, an entity name and a user-readable identifier (`Identifiable#getIdentifier`). `BaseOBObject#toString` includes the non-null values of the identifier properties, `DalUtil#sortByIdentifier(List)` sorts objects by the identifier, and `DalUtil#getValueFromPath` returns it for the path part `_identifier` (`BaseOBObject#toString`, `DalUtil#sortByIdentifier`, `DalUtil#getValueFromPath`).

Used in: [02-runtime-model](./02-runtime-model.md#identity-and-properties)

### Inherited access

Inherited access is an entity flag that `Property#initializeName` sets when the entity has a property named `inheritedFrom` that references the `ADRole` entity (`Property#initializeName`). When `Entity#isInheritedAccessEnabled` is true, `Entity#getImplementsStatement` adds `InheritedAccessEnabled` to the `implements` clause of the [generated class](#generated-class); none of the DAL classes cited in 02 tests for that interface (`Entity#getImplementsStatement`).

Used in: [02-runtime-model](./02-runtime-model.md#flags)

### Interceptor

An interceptor is an object whose callbacks Hibernate invokes when a session saves, flushes or deletes objects: `DalSessionFactoryController#setInterceptor` installs a new `OBInterceptor` on the Hibernate `Configuration` it receives (`DalSessionFactoryController#setInterceptor`). `OBInterceptor#doEvent`, called from `OBInterceptor#onSave` and `OBInterceptor#onFlushDirty`, sets the [audit properties](#audit-properties) of a `Traceable` object, sets a null client and organization of a new object to proxies of the current ones through `OBInterceptor#onNew`, and runs `SecurityChecker#checkWriteAccess(Object)` (`OBInterceptor#doEvent`, `OBInterceptor#onNew`). `OBInterceptor#onDelete` runs `SecurityChecker#checkDeleteAllowed(Object)`, `OBInterceptor#onFlushDirty` also runs the [cross-organization reference check](#cross-organization-reference-check), and `OBInterceptor#postFlush` clears the [new object](#new-object) flag (`OBInterceptor#onDelete`, `OBInterceptor#onFlushDirty`, `OBInterceptor#postFlush`). The [model session factory](#model-session-factory) installs its own `LocalInterceptor`, which fails on every save, delete, dirty flush and collection change (`ModelSessionFactoryController#setInterceptor`).

Used in: [01-architecture](./01-architecture.md#layers), [02-runtime-model](./02-runtime-model.md#how-the-runtime-model-is-built), [04-security-and-filtering](./04-security-and-filtering.md#reading-guide)

### Many-to-one

A many-to-one is the Hibernate reference mapping that `DalMappingGenerator#generateReferenceMapping` emits for every [reference property](#reference-property) with a target entity that is not [one-to-one](#one-to-one): it carries the target `entity-name`, a `column`, or a `formula` when SQL logic is set, `not-null="true"` when the property is mandatory, `update="false"` and `insert="false"` when the property is inactive or belongs to a [view entity](#view-entity), and a [property-ref](#property-ref) when the referenced property is not the target's id (`DalMappingGenerator#generateReferenceMapping`). `DalMappingGenerator#generateCompositeID` uses a `<key-many-to-one>` for each reference part of a [composite id](#composite-id), and `DalMappingGenerator#generateComputedColumnsMapping` maps the `_computedColumns` property as a many-to-one (`DalMappingGenerator#generateCompositeID`, `DalMappingGenerator#generateComputedColumnsMapping`).

Used in: [01-architecture](./01-architecture.md#per-entity-rules)

### Model session factory

The model session factory is the Hibernate session factory of a `ModelSessionFactoryController`, which `ModelProvider#initialize` creates to read the [Application Dictionary](#application-dictionary-ad) instead of using the DAL [session factory](#session-factory), and closes in a `finally` block (`ModelProvider#initialize`). It maps the eight AD model classes `Column`, `Module`, `Package`, `Reference`, `RefList`, `RefSearch`, `RefTable` and `Table`, plus every class added through `ModelSessionFactoryController#addAdditionalClasses` (`ModelSessionFactoryController#mapModel`). Its `LocalInterceptor` fails with `Check.fail` on every save, delete, dirty flush and collection remove, recreate or update, so the model session is read-only (`ModelSessionFactoryController#setInterceptor`). The source comment in `ModelProvider#initialize` gives the reason for a separate factory: the DAL layer uses `ModelProvider`, so reading the model through the DAL would create a cyclic relation; the same comment speaks of using `SessionHandler` directly, while the code creates a `ModelSessionFactoryController` (`ModelProvider#initialize`). `ModelProvider#computeLastUpdateModelTime` also opens a session on a new `ModelSessionFactoryController` and closes it afterwards (`ModelProvider#computeLastUpdateModelTime`).

Used in: [01-architecture](./01-architecture.md#startup-sequence), [02-runtime-model](./02-runtime-model.md#how-the-runtime-model-is-built)

### Named parameter

A named parameter is a `:name` placeholder in an HQL clause whose value is bound by name: `OBQuery#setNamedParameter(String, Object)` puts one value into the parameter map of the query, and when the Hibernate query is created, `OBQuery#setParameters(Query)` binds a `Collection` or `String[]` value as a parameter list and any other value as a single parameter (`OBQuery#setNamedParameter(String, Object)`, `OBQuery#setParameters(Query)`). `OBDal#createQuery(Class, String, Map)` takes a whole map, and `OBQuery#setNamedParameters(Map)` stores a map without copying it (`OBDal#createQuery(Class, String, Map)`, `OBQuery#setNamedParameters(Map)`). `OBQuery#addOrgClientActiveFilter` binds the [readable organizations](#readable-organizations) and [readable clients](#readable-clients) as the named parameters `_dal_readableOrganizations_dal_` and `_dal_readableClients_dal_` (`OBQuery#addOrgClientActiveFilter`). `OBQuery#setParameters(List)` rewrites each [positional parameter](#positional-parameter) as a named one (`OBQuery#setParameters(List)`).

Used in: [03-dal-service-api](./03-dal-service-api.md#obdalcreatequeryclass-string-list), [04-security-and-filtering](./04-security-and-filtering.md#query-restrictions)

### Natural tree

The natural tree of an organization is the organization together with its parent tree and its child tree, as `OrganizationStructureProvider#getNaturalTree(String)` returns it, or a set that holds only the passed id when the provider has no node for it (`OrganizationStructureProvider#getNaturalTree(String)`). `OBContext#setReadableOrganizations(Role)` builds the [readable organizations](#readable-organizations) as the union of the natural trees of the active organizations of the role, unless those organizations include `"0"` (`OBContext#setReadableOrganizations(Role)`). `OrganizationStructureProvider#isInNaturalTree(Organization, Organization)` is true when either id is `"0"`, and otherwise when the second organization is in the natural tree of the first, and the [cross-organization reference check](#cross-organization-reference-check) uses it (`OrganizationStructureProvider#isInNaturalTree`, `OBInterceptor#checkReferencedOrganizations`).

Used in: [04-security-and-filtering](./04-security-and-filtering.md#readable-and-writable-sets)

### New object

A new object is a [business object](#business-object) for which `BaseOBObject#isNewOBObject` is true, because its id is null or because `BaseOBObject#setNewOBObject` has set the new-object flag (`BaseOBObject#isNewOBObject`). The field comment gives the reason for the flag: it forces an insert of the object, which is useful when the id of an imported object should be preserved (`BaseOBObject#setNewOBObject`). `OBInterceptor#isTransient` returns true for a `BaseOBObject` whose id is set and whose flag is set, and null otherwise, which leaves the decision to Hibernate (`OBInterceptor#isTransient`). `OBInterceptor#postFlush` sets the flag back to false for every flushed object, and `OBInterceptor#onFlushDirty` logs a warning for an object without previous state, a case the warning attributes to an id set without `setNewObject(true)` (`OBInterceptor#postFlush`, `OBInterceptor#onFlushDirty`).

Used in: [02-runtime-model](./02-runtime-model.md#values-identity-and-the-new-object-flag), [04-security-and-filtering](./04-security-and-filtering.md#callbacks)

### One-to-many property

A one-to-many property is a property whose value is a list, flagged by `Property#isOneToMany`; in the model that `ModelProvider#initialize` builds, these properties are created by `ModelProvider#createChildProperty` on the parent entity, not from a column (`Property#isOneToMany`, `ModelProvider#createChildProperty`). `DalMappingGenerator#generateOneToMany` maps each one as an inverse [bag](#bag) keyed on the column of the referenced property (`DalMappingGenerator#generateOneToMany`). `Property#getFormattedDefaultValue` gives it the default `new ArrayList<Object>()` in the generated class (`Property#getFormattedDefaultValue`). `DalUtil#copy(BaseOBObject, boolean, boolean, Map)` copies a one-to-many list only when children are copied, the property is a child and its target entity is not a view (`DalUtil#copy(BaseOBObject, boolean, boolean, Map)`).

Used in: [01-architecture](./01-architecture.md#per-entity-rules), [02-runtime-model](./02-runtime-model.md#how-the-runtime-model-is-built)

### One-to-one

A one-to-one is the Hibernate reference mapping that `DalMappingGenerator#generateReferenceMapping` emits for a one-to-one property as `<one-to-one constrained="true">`, named after the simple name of its type with a lower-case first letter, with the target `entity-name` and a [property-ref](#property-ref) when the referenced property is not the target's id (`DalMappingGenerator#generateReferenceMapping`). `ModelProvider#createIdReferenceProperty` creates such a property for an entity with one non-primitive id property, as the reference half of the two mappings that an id which is also a [foreign key](#foreign-key) needs (`ModelProvider#createIdReferenceProperty`, `ModelProvider#initialize`).

Used in: [01-architecture](./01-architecture.md#per-entity-rules), [02-runtime-model](./02-runtime-model.md#how-the-runtime-model-is-built)

### Org/client access check

The org/client access check is the client and writable-organization part of `SecurityChecker#checkWriteAccess(Object)`, which runs, for an object that has a client, when the context is outside admin mode or `OBContext#doOrgClientAccessCheck()` is true (`SecurityChecker#checkWriteAccess`). `OBContext#doOrgClientAccessCheck()` is false when the top of the [admin mode stack](#admin-mode-stack) carries a false org/client flag, as `OBContext#setAdminMode()` and `OBContext#setAdminMode(boolean)` with `false` push it, or when the deprecated admin flag is set or the role is `"0"`, and true in every other case (`OBContext#doOrgClientAccessCheck()`). In admin mode `OBDal#save(Object)` records the value of `OBContext#doOrgClientAccessCheck()` on the object through `BaseOBObject#setAccessChecks(boolean, boolean)`, and `SecurityChecker#checkWriteAccess` later skips the writable-organization test for an object whose flag is false (`OBDal#save(Object)`, `SecurityChecker#checkWriteAccess`).

Used in: [03-dal-service-api](./03-dal-service-api.md#obdalsaveobject), [04-security-and-filtering](./04-security-and-filtering.md#mode-predicates)

### Organization

An organization is the record that a business object of an organization-enabled entity references through its property named `organization`; `Property#initializeName` sets the entity's organization-enabled flag when it finds that property, and also the organization-part-of-key flag when the property is an id or part of a [composite id](#composite-id) (`Property#initializeName`). `OBContext#initialize(String, String, String, String, String, String)` resolves the current organization of the [user context](#user-context), and `OBContext` computes the [readable organizations](#readable-organizations), [writable organizations](#writable-organizations) and [deactivated organizations](#deactivated-organization) from the [role](#role) (`OBContext#initialize(String, String, String, String, String, String)`, `OBContext#getReadableOrganizations()`). When the context is outside admin mode or the [org/client access check](#orgclient-access-check) is on, `SecurityChecker#checkWriteAccess` requires the organization of an object to be among the writable organizations, with three exceptions, and `OBDal#save(Object)` fills a missing organization with a [proxy](#proxy) of the current organization (`SecurityChecker#checkWriteAccess`, `OBDal#setClientOrganization`).

- Code qualification: filtering by organization is applied by `OBCriteria#initialize` and `OBQuery#addOrgClientActiveFilter`, and only for organization-enabled entities or entities whose organization is part of the key, which are restricted on `id.organization.id` instead of `organization.id` (`OBCriteria#initialize`, `OBQuery#addOrgClientActiveFilter`). The filter can be switched off per query with `OBCriteria#setFilterOnReadableOrganization(boolean)` or `OBQuery#setFilterOnReadableOrganization(boolean)` (`OBCriteria#setFilterOnReadableOrganization(boolean)`, `OBQuery#setFilterOnReadableOrganization(boolean)`). It is added in admin mode too, because only the read check of these methods depends on `OBContext#isInAdministratorMode()` (`OBCriteria#initialize`). `OBDal#get(Class, Object)` and `OBDal#get(String, Object)` add no row filter and perform only the entity-level read check (`OBDal#get(Class, Object)`, `OBDal#get(String, Object)`).

Used in: [01-architecture](./01-architecture.md#per-entity-rules), [02-runtime-model](./02-runtime-model.md#access-level), [03-dal-service-api](./03-dal-service-api.md#reading-guide), [04-security-and-filtering](./04-security-and-filtering.md#reading-guide)

### Pool

A pool is a named database connection pool for which the DAL keeps its own session and transaction: a `SessionHandler` holds one Hibernate [session](#session), one [transaction](#transaction) and an optional connection per pool name, and `SessionHandler#getSession(String)` treats a null name as the default pool `ExternalConnectionPool.DEFAULT_POOL` (boundary) (`SessionHandler#getSession(String)`). `OBDal#getInstance()` returns the instance for the default pool, `OBDal#getReadOnlyInstance()` the one for `ExternalConnectionPool.READONLY_POOL`, and `OBDal#getInstance(String)` the instance kept for a pool name, or the default instance when `DataPoolChecker` (boundary) reports that the default pool must be used (`OBDal#getInstance()`, `OBDal#getReadOnlyInstance()`, `OBDal#getInstance(String)`). When the `db.externalPoolClassName` setting is set, `SessionHandler#createSession(String)` opens a pool's session with a connection from the external connection pool (`SessionHandler#createSession(String)`).

Used in: [01-architecture](./01-architecture.md#request-lifecycle), [03-dal-service-api](./03-dal-service-api.md#reading-guide)

### Positional parameter

A positional parameter is a `?` marker in a query clause whose value is bound by its position: `OBQuery#setParameters(List)` rewrites each `?` of the stored clause as the [named parameter](#named-parameter) `:__p0`, `:__p1` and so on, sets the value at that position, and fails with `Check.isTrue` when the clause holds fewer markers than the list holds values (`OBQuery#setParameters(List)`). Its Javadoc gives the reason, that legacy-style query parameters are no longer supported in Hibernate, and names `setNamedParameters(Map)` as the replacement (`OBQuery#setParameters(List)`). The deprecated `OBDal#createQuery(Class, String, List)` hands its list to that method (`OBDal#createQuery(Class, String, List)`).

Used in: [03-dal-service-api](./03-dal-service-api.md#obdalcreatequeryclass-string-list)

### Primitive property

A primitive property is a property whose domain type is a `PrimitiveDomainType`: `Property#isPrimitive` reports it, and `Property#getPrimitiveType` then gives its Java type (`Property#isPrimitive`, `Property#getPrimitiveType`). `DalMappingGenerator#generatePrimitiveMapping` emits nothing for one whose Hibernate type is `Object`, maps booleans with Hibernate's `YesNoType`, uses a `formula` instead of a `column` when SQL logic is set, adds `not-null="true"` when it is mandatory, and sets `update="false"` and `insert="false"` for inactive properties, properties of [view entities](#view-entity) and properties with SQL logic (`DalMappingGenerator#generatePrimitiveMapping`).

Used in: [01-architecture](./01-architecture.md#per-entity-rules), [02-runtime-model](./02-runtime-model.md#how-the-runtime-model-is-built)

### Property

A property is the runtime-model object of class `Property` that models one attribute of an [entity](#entity); its Javadoc says a property can be a primitive type, a reference or a list (one-to-many) property (`Property#isPrimitive`, `Property#isOneToMany`). `Property#initializeFromColumn` copies from the AD column the key flag as id, the identifier and parent flags, the column name, the domain type, the default value, the mandatory flag and the other column settings, and `Property#initializeName` gives it its name (`Property#initializeFromColumn`, `Property#initializeName`). `Entity#getProperties` lists the properties of an entity, and `Property#getIndexInEntity` gives a property's position in that list, which indexes the value array of `BaseOBObject` (`Entity#getProperties`, `Property#getIndexInEntity`). The mandatory flag is copied from the column and then overwritten by `ModelProvider#initialize` from the database not-null value, except for the entities and properties that 02 lists (`ModelProvider#initialize`).

Used in: [01-architecture](./01-architecture.md#layers), [02-runtime-model](./02-runtime-model.md#reading-guide), [03-dal-service-api](./03-dal-service-api.md#obdalfinduniqueconstrainedobjectsbaseobobject), [04-security-and-filtering](./04-security-and-filtering.md#access-checks)

### Property-ref

A property-ref is the attribute of a Hibernate reference mapping that names the target property when the reference does not point to the target's id: `DalMappingGenerator#generateReferenceMapping` adds it to [one-to-one](#one-to-one) and [many-to-one](#many-to-one) mappings when the referenced property is not the target's id (`DalMappingGenerator#generateReferenceMapping`). `Property#setReferencedProperty` records the referenced property of a [reference property](#reference-property) and sets the target entity to that property's entity (`Property#setReferencedProperty`).

Used in: [01-architecture](./01-architecture.md#per-entity-rules)

### Provider registration

A provider registration is an entry that `OBProvider` keeps under a class name or service name, recording the implementation class to instantiate, whether its instance is a [singleton](#singleton) and whether the registration can be overwritten (`OBProvider#register(String, Class, boolean)`). `OBProvider#register(Class, Class, boolean)` registers `instanceClass` as the implementation of the Openbravo class `registrationClass`, and per its Javadoc overwrites a current registration only when `overwrite` is true; it is the extension point for supplying a different implementation of an Openbravo class (`OBProvider#register(Class, Class, boolean)`). A registration created with `overwrite` true is itself not overwritable, and the source comment gives the reason: a registration which overwrites others is not overwritable (`OBProvider#register(String, Class, boolean)`). `OBProvider#get(Class)` returns the instance of the registration for a class, registering the class as its own implementation first when no registration exists, and `OBProvider#registerInstance(Class, Object, boolean)` sets a fixed instance on a registration (`OBProvider#get(Class)`, `OBProvider#registerInstance(Class, Object, boolean)`).

Used in: [03-dal-service-api](./03-dal-service-api.md#reading-guide)

### Proxy

A proxy is a Hibernate placeholder for an object that is not loaded yet: `OBDal#getProxy(String, Object)` returns one through `internalLoad` with no read-access check and no existence check, and its Javadoc states that a missing row is detected only when a referencing object is persisted or the proxy is initialized (`OBDal#getProxy(String, Object)`). `OBDal#save(Object)` fills a missing client or organization with proxies of the current ones, and `OBInterceptor#onNew` sets the audit users, the client and the organization to proxies obtained through `OBDal#getProxy(Class, String)` (`OBDal#setClientOrganization`, `OBInterceptor#onNew`). The deprecated `DalUtil#getId` and `DalUtil#getEntityName(Object)` read the id and the entity name of a Hibernate proxy without loading the object (`DalUtil#getId`, `DalUtil#getEntityName(Object)`). In the runtime model the word also names the `_computedColumns` property, which `Property#isProxy` marks and whose mandatory flag `ModelProvider#initialize` does not log as missing (`Property#isProxy`, `ModelProvider#initialize`).

Used in: [02-runtime-model](./02-runtime-model.md#how-the-runtime-model-is-built), [03-dal-service-api](./03-dal-service-api.md#obdalsaveobject), [04-security-and-filtering](./04-security-and-filtering.md#shared-event-handling)

### Readable clients

The readable clients are the client ids to which `OBCriteria` and `OBQuery` restrict queries on client-enabled entities: `OBContext#setReadableClients(Role)` sets them to `{"0"}` when the [user level](#user-level) is `S` or the client of the role is `"0"`, and otherwise to the client of the role plus `"0"` (`OBContext#setReadableClients(Role)`). `OBContext#getReadableClients()` computes the array when none is cached and returns a copy (`OBContext#getReadableClients()`). With the client switch on, `OBCriteria#initialize` restricts `client.id` to them, and `OBQuery#addOrgClientActiveFilter` binds them as the [named parameter](#named-parameter) `_dal_readableClients_dal_` (`OBCriteria#initialize`, `OBQuery#addOrgClientActiveFilter`). The deprecated `OBDal#getReadableClientsInClause()` returns them as an HQL in-clause (`OBDal#getReadableClientsInClause()`).

Used in: [03-dal-service-api](./03-dal-service-api.md#obdalgetreadableclientsinclause), [04-security-and-filtering](./04-security-and-filtering.md#reading-guide)

### Readable organizations

The readable organizations are the organization ids to which `OBCriteria` and `OBQuery` restrict queries on organization-enabled entities: `OBContext#setReadableOrganizations(Role)` computes them as every organization of the current client when the active organizations of the role include `"0"`, and otherwise as the union of the [natural trees](#natural-tree) of those organizations, and always adds `"0"` (`OBContext#setReadableOrganizations(Role)`). With the organization switch on, `OBCriteria#initialize` restricts `organization.id`, or `id.organization.id` when the organization is part of the key, to them, and `OBQuery#addOrgClientActiveFilter` binds them as the [named parameter](#named-parameter) `_dal_readableOrganizations_dal_` (`OBCriteria#initialize`, `OBQuery#addOrgClientActiveFilter`). `OBContext#getReadableOrganizations()` computes them when none are cached, which is the case on first use and after `OBContext#setRole(Role)` or `OBContext#addWritableOrganization(String)` has cleared them, and returns a copy (`OBContext#getReadableOrganizations()`). The deprecated `OBDal#getReadableOrganizationsInClause()` returns them as an HQL in-clause (`OBDal#getReadableOrganizationsInClause()`).

Used in: [03-dal-service-api](./03-dal-service-api.md#obdalgetreadableorganizationsinclause), [04-security-and-filtering](./04-security-and-filtering.md#reading-guide)

### Reference property

A reference property is a property whose value is an object of another entity: for each non-primitive column, `ModelProvider#setReferencedPropertiesForTable` links the column's property to the property of the referenced column through `Property#setReferencedProperty`, which also sets the target entity and flags the referenced property as being referenced (`ModelProvider#setReferencedPropertiesForTable`, `Property#setReferencedProperty`). `ModelProvider#setReferenceProperties` runs that pass for the tables of the model, and `ModelProvider#setVirtualPropertiesForReferenceId` adds a [one-to-one](#one-to-one) reference property for an id that is a reference (`ModelProvider#setReferenceProperties`, `ModelProvider#setVirtualPropertiesForReferenceId`). `DalMappingGenerator#generateReferenceMapping` maps it as a one-to-one or a [many-to-one](#many-to-one) (`DalMappingGenerator#generateReferenceMapping`).

Used in: [01-architecture](./01-architecture.md#per-entity-rules), [02-runtime-model](./02-runtime-model.md#how-the-runtime-model-is-built)

### Role

A role is the record under which the [user context](#user-context) computes its access: `OBContext#initialize(String, String, String, String, String, String)` takes the passed role id, otherwise the default role of the user when that role is active, otherwise the first active user-role assignment with an active role, and throws `OBSecurityException` when there is none (`OBContext#initialize(String, String, String, String, String, String)`). `OBContext#setRole(Role)` marks the context as administrator when the role id is `"0"`, takes the [user level](#user-level) from the role, and discards the cached `EntityAccessChecker` and every cached set, so that they are recomputed from the new role on next use (`OBContext#setRole(Role)`). The role determines the [readable clients](#readable-clients), the [readable organizations](#readable-organizations), the [writable organizations](#writable-organizations) and, through `OBContext#getEntityAccessChecker`, the [entity access](#entity-access) of the context (`OBContext#setReadableClients(Role)`, `OBContext#getEntityAccessChecker`). Role `"0"` puts the context in admin mode and switches the [org/client access check](#orgclient-access-check) off whatever the admin mode stack holds (`OBContext#isInAdministratorMode()`, `OBContext#doOrgClientAccessCheck()`).

Used in: [04-security-and-filtering](./04-security-and-filtering.md#reading-guide)

### Runtime model

The runtime model is the in-memory list of `Entity` objects, each with its `Property` objects, that `ModelProvider#getModel` returns and that the Hibernate mapping, the entity generator and the DAL services read (`ModelProvider#getModel`, `DalMappingGenerator#generateMapping()`, `GenerateEntitiesTask#execute`). `ModelProvider#initialize` builds it from the [Application Dictionary](#application-dictionary-ad), and the lookups `ModelProvider#getEntity(String)`, `ModelProvider#getEntity(Class)`, `ModelProvider#getEntityByTableName` and `ModelProvider#getEntityByTableId` read the indexes it fills (`ModelProvider#initialize`). `ModelProvider#getEntity(String)` throws `CheckException` with "not found in runtime model" for an unknown entity name (`ModelProvider#getEntity(String)`).

- Code qualification: it is built lazily on the first `ModelProvider#getModel()` call and rebuilt by `ModelProvider#refresh()`, which replaces the registered `ModelProvider` with a new instance and builds its model (`ModelProvider#getModel`, `ModelProvider#refresh`). It is also built at [development time](#development-time) by `GenerateEntitiesTask#execute`, which calls `ModelProvider#getModel` (`GenerateEntitiesTask#execute`). Its mandatory flags come from the database not-null metadata that `ModelProvider#getColumnMandatories` reads, not only from the AD column flag (`ModelProvider#getColumnMandatories`, `ModelProvider#initialize`).

Used in: [01-architecture](./01-architecture.md#reading-guide), [02-runtime-model](./02-runtime-model.md#reading-guide), [03-dal-service-api](./03-dal-service-api.md#obdal-creating-queries)

### Session

A session is a Hibernate session, through which the DAL loads and saves objects: `SessionHandler#getSession(String)` returns the session that the thread's [session handler](#session-handler) holds for a [pool](#pool), and begins a session and a [transaction](#transaction) on first use (`SessionHandler#getSession(String)`). `SessionHandler#begin(String)` creates it with the [flush mode](#flush-mode) `FlushMode.COMMIT`, and `SessionHandler#createSession(String)` opens it from the session factory of `SessionFactoryController` (boundary) (`SessionHandler#begin(String)`, `SessionHandler#createSession(String)`). `OBDal#getSession()` returns the session of the instance's pool, and `SessionHandler#commitAndClose(String)` and `SessionHandler#rollback(String)` close it (`OBDal#getSession()`, `SessionHandler#commitAndClose(String)`, `SessionHandler#rollback(String)`). The session that `ModelProvider#initialize` opens on the [model session factory](#model-session-factory) is separate and read-only (`ModelProvider#initialize`, `ModelSessionFactoryController#setInterceptor`).

Used in: [01-architecture](./01-architecture.md#reading-guide), [02-runtime-model](./02-runtime-model.md#how-the-runtime-model-is-built), [03-dal-service-api](./03-dal-service-api.md#reading-guide), [04-security-and-filtering](./04-security-and-filtering.md#reading-guide)

### Session factory

A session factory is the Hibernate object that opens sessions: `DalSessionFactory` delegates to the Hibernate session factory passed to `DalSessionFactory#setDelegateSessionFactory`, and `DalSessionFactory#openSession()` initializes the database [session info](#session-info) on the connection of the session it opens (`DalSessionFactory#setDelegateSessionFactory`, `DalSessionFactory#openSession()`). Its class Javadoc gives the reason for the wrapper: every call is delegated to the real session factory except the calls that open a session, which first set session information in the database; the Javadoc of `DalSessionFactory#getDelegateSessionFactory` adds that normal application code must use `DalSessionFactory` (`DalSessionFactory#openSession()`, `DalSessionFactory#getDelegateSessionFactory`). The `DalSessionFactoryController` class Javadoc describes that controller as the class that initializes and provides the session factory of the runtime DAL, and `DalSessionFactoryController#setInterceptor` installs `OBInterceptor` on its configuration (`DalSessionFactoryController#mapModel`, `DalSessionFactoryController#setInterceptor`). The Application Dictionary is read through a separate [model session factory](#model-session-factory) (`ModelProvider#initialize`).

Used in: [01-architecture](./01-architecture.md#reading-guide), [02-runtime-model](./02-runtime-model.md#how-the-runtime-model-is-built)

### Session handler

The session handler is the `SessionHandler` of the current thread, which keeps one Hibernate session, one transaction and an optional connection per [pool](#pool): `SessionHandler#getInstance` returns the handler stored in a thread-local and, when the thread has none, creates one through `OBProvider#get(Class)` (`SessionHandler#getInstance`). `SessionHandler#deleteSessionHandler` removes it from the thread-local, so the next `SessionHandler#getInstance` creates a new one, and `SessionHandler#isSessionHandlerPresent(String)` tells whether the thread has a handler available for a pool (`SessionHandler#deleteSessionHandler`, `SessionHandler#isSessionHandlerPresent(String)`). `OBDal#commitAndClose()` and `OBDal#rollbackAndClose()` act only when that method returns true for their pool (`OBDal#commitAndClose()`, `OBDal#rollbackAndClose()`).

Used in: [01-architecture](./01-architecture.md#one-handler-per-thread)

### Session info

Session info is the user session information that the DAL sets in the database on a connection: `DalSessionFactory#openSession()` calls `DalSessionFactory#initConnection`, which initializes it and then runs the statement configured in the `bbdd.sessionConfig` setting, while `DalSessionFactory#openStatelessSession()` and `DalSessionFactory#openStatelessSession(Connection)` only initialize it (`DalSessionFactory#openSession()`, `DalSessionFactory#openStatelessSession()`). No source states whether that difference is intended, and 01 records it as an ambiguity (`DalSessionFactory#openSession()`). In the `finally` block of its end-of-request hook, `DalRequestFilter#doFilter` calls `SessionInfo.init()` (boundary), which its source comment describes as setting all session info to null (`DalRequestFilter#doFilter`). `OBDal#flush()` passes its connection to `SessionInfo.saveContextInfoIntoDB` (boundary) before it flushes a dirty session (`OBDal#flush()`).

Used in: [01-architecture](./01-architecture.md#startup-sequence)

### Session-in-view

Session-in-view is the pattern that the Javadoc of `SessionHandler#doSessionInViewPatter` defines as closing and committing the session at the end of the request, and the method always returns true (`SessionHandler#doSessionInViewPatter`). The `DalRequestFilter` class Javadoc states that each request is handled within a transaction that is committed or rolled back at the end of the request (`DalRequestFilter#doFilter`).

Used in: [01-architecture](./01-architecture.md#session-in-view)

### Singleton

A singleton is a class of which one shared instance is used: `OBProvider#register(String, Class, boolean)` makes a [provider registration](#provider-registration) a singleton when the instance class implements `OBSingleton` (boundary), and the Javadoc of `OBProvider#removeInstance(Class)` describes the singleton instance of a class as held in the internal registry and recreated at the next request after its removal (`OBProvider#register(String, Class, boolean)`, `OBProvider#removeInstance(Class)`). The `OBProvider` class Javadoc describes it as an implementation of the service locator pattern in which each registered class is treated as a singleton or not (`OBProvider#register(String, Class, boolean)`). `OBDal` implements `OBNotSingleton`, whose Javadoc nevertheless reads "Tags a class as being a singleton", and `OBDal#getInstance()` lazily keeps the instance it obtains from `OBProvider#get(Class)` in a static field whenever that field is null; 02 records this as an ambiguity (`OBDal#getInstance()`, `OBProvider#register(String, Class, boolean)`). The null check in `OBDal#getInstance()` is not synchronized, so the code makes no exactly-once or thread-safety promise for that instance (`OBDal#getInstance()`).

Used in: [02-runtime-model](./02-runtime-model.md#implemented-interfaces), [03-dal-service-api](./03-dal-service-api.md#obprovider)

### SQL function

An SQL function is an entry of the map of `SQLFunction` objects that `DalSessionFactoryController#getSQLFunctions` returns: the method merges the maps returned by every injected `SQLFunctionRegister` (boundary), skips registers that return null, and caches the merged map for later calls (`DalSessionFactoryController#getSQLFunctions`). `DalSessionFactoryController` overrides this method of its boundary superclass `SessionFactoryController`, so which code calls it, and when, is decided inside the boundary classes `DalLayerInitializer` and `SessionFactoryController` (`DalSessionFactoryController#mapModel`).

Used in: [01-architecture](./01-architecture.md#registering-the-mapping)

### src-gen

`src-gen` is the source folder, set by the `base.src.gen` property of `build.xml`, into which `GenerateEntitiesTask#execute` writes the [generated classes](#generated-class), one UTF-8 file per entity at `src-gen/<package path>/<Class>.java` (`GenerateEntitiesTask#execute`). `src/build.xml#clean.src.gen` deletes its contents except files matching `**/.keep`, and `GenerateEntitiesTask#hasChanged` is true when `src-gen/org/openbravo/model/ad` does not exist (`src/build.xml#clean.src.gen`, `GenerateEntitiesTask#hasChanged`). In the repository the folder holds only the empty `src-gen/.keep`, so no generated class is available, and `SystemInformation` is recorded as Not Found (`GenerateEntitiesTask#execute`).

Used in: [01-architecture](./01-architecture.md#entity-generation-generateentities), [02-runtime-model](./02-runtime-model.md#worked-example-systeminformation)

### Stateless session

A stateless session is a Hibernate session that `DalSessionFactory#openStatelessSession()` or `DalSessionFactory#openStatelessSession(Connection)` opens; both carry the Javadoc sentence that they set user session information in the database, but only initialize [session info](#session-info) and do not run the `bbdd.sessionConfig` statement that `DalSessionFactory#openSession()` runs (`DalSessionFactory#openStatelessSession()`). No source states whether the difference is intended, and 01 records it as an ambiguity that it does not resolve (`DalSessionFactory#openSession()`).

Used in: [01-architecture](./01-architecture.md#startup-sequence)

### Thread handler

A thread handler is a `DalThreadHandler` (boundary), inside which `DalRequestFilter#doFilter` runs each request: it creates an anonymous subclass that overrides the hooks `doBefore`, `doAction` and `doFinal`, and calls its `run()` (`DalRequestFilter#doFilter`). The `DalRequestFilter` class Javadoc gives the reason: handling the request thread inside a `DalThreadHandler` ensures that every request is handled within a transaction that is committed or rolled back at the end of the request (`DalRequestFilter#doFilter`). The Javadoc of `SessionHandler#setDoRollback(boolean)` says the `DalThreadHandler` uses the rollback request it records, while which cleanup calls the handler makes, and in what order, is not shown by any in-scope code (`SessionHandler#setDoRollback(boolean)`, `DalRequestFilter#doFilter`).

Used in: [01-architecture](./01-architecture.md#request-lifecycle)

### Transaction

A transaction is the database transaction of a DAL session: `SessionHandler#begin(String)` begins one when it creates the session of a [pool](#pool), and `SessionHandler#commitAndClose(String)` calls `SessionHandler#flushRemainingChanges`, commits the active transaction and closes the connection and the session, rolling back and closing instead when a step fails (`SessionHandler#begin(String)`, `SessionHandler#commitAndClose(String)`). `SessionHandler#rollback(String)` rolls it back and closes the session, and `SessionHandler#commitAndStart()` commits and begins a new transaction on the same session (`SessionHandler#rollback(String)`, `SessionHandler#commitAndStart()`). `SessionHandler#setDoRollback(boolean)` records that the transaction is to be rolled back at the end of the thread, and `SecurityChecker#checkWriteAccess` requests this before it throws for a client or organization failure (`SessionHandler#setDoRollback(boolean)`, `SecurityChecker#checkWriteAccess`). Per the `DalRequestFilter` class Javadoc, the transaction of a request ends with the request, committed or rolled back (`DalRequestFilter#doFilter`).

Used in: [01-architecture](./01-architecture.md#reading-guide), [02-runtime-model](./02-runtime-model.md#how-the-runtime-model-is-built), [03-dal-service-api](./03-dal-service-api.md#reading-guide), [04-security-and-filtering](./04-security-and-filtering.md#write-access-check)

### Translatable property

A translatable property is a property whose values can be read in another language from a translation entity: `ModelProvider#setTranslatableColumns` looks up the entity of the table named after the column's table plus `_Trl` and passes its property for the same column to `Property#setTranslatable` (`ModelProvider#setTranslatableColumns`). `Property#setTranslatable` makes the property translatable only when the translation entity has an `ad_language` column, the entity has a one-to-many property whose target is the translation entity, and a parent property of the translation entity references the entity's first id property; otherwise it logs a warning and `Property#isTranslatable` stays false (`Property#setTranslatable`). With a language, and while `OBContext#hasTranslationInstalled` is true, `BaseOBObject#get(String, Language, String)` looks the translation record up at most once per object, through an `OBCriteria` with the client and organization filters off, and returns the value of the translation property when one is found (`BaseOBObject#get(String, Language, String)`).

Used in: [02-runtime-model](./02-runtime-model.md#translatable-properties), [04-security-and-filtering](./04-security-and-filtering.md#reads-by-id-and-translation-lookup)

### Typed API

The typed API is the set of getters and setters of the [generated classes](#generated-class), used together with class-based calls such as `OBDal#get(Class, Object)`: `Property#getGetterSetterName` gives the accessor name, dropping the `is` prefix of a boolean property name, and `Property#getTypeName` and `Property#getObjectTypeName` give its type (`Property#getGetterSetterName`, `Property#getTypeName`, `OBDal#get(Class, Object)`). `Entity#getClassName` names the generated class, and the public static `ENTITY_NAME` field that `DalUtil#getEntityName(Class)` reads gives its [entity name](#entity-name) (`Entity#getClassName`, `DalUtil#getEntityName(Class)`). The concrete generated members, and the checks the generated accessors perform, cannot be determined from the repository, because the generated sources are Not Found (`GenerateEntitiesTask#execute`). The alternative is the [dynamic API](#dynamic-api), which needs only the entity name and the runtime model (`BaseOBObject#get(String)`, `Entity#getMappingClass`).

Used in: [01-architecture](./01-architecture.md#entity-generation-generateentities), [02-runtime-model](./02-runtime-model.md#reading-guide)

### Unique constraint

A unique constraint is a database constraint over a set of columns whose combined values must be unique: `ModelProvider#buildUniqueConstraints` reads every unique constraint from the database through a native query and skips one whose table has no table-based entity (`ModelProvider#buildUniqueConstraints`). `OBDal#findUniqueConstrainedObjects(BaseOBObject)` takes the unique constraints of the object's entity from `Entity#getUniqueConstraints`, and for each one looks for other objects with the same values, excluding the object's own id when it is not null (`OBDal#findUniqueConstrainedObjects(BaseOBObject)`).

Used in: [02-runtime-model](./02-runtime-model.md#how-the-runtime-model-is-built), [03-dal-service-api](./03-dal-service-api.md#obdalfinduniqueconstrainedobjectsbaseobobject)

### User context

The user context is the `OBContext` of the current thread, which holds the user, [role](#role), client, organization, language and warehouse, and the readable and writable sets computed from them (`OBContext#initialize(String, String, String, String, String, String)`). `OBContext#getOBContext()` returns it, or null when none is set, and `OBContext#setOBContext(OBContext)` stores it in a static thread-local field (`OBContext#getOBContext()`, `OBContext#setOBContext(OBContext)`). `DalRequestFilter#doFilter` sets it for each request through `OBContext#setOBContext(HttpServletRequest)`, and module code installs one for known ids with an `OBContext#setOBContext(String, String, String, String)` overload (`DalRequestFilter#doFilter`, `OBContext#setOBContext(String, String, String, String)`). `OBContext#createOBContext(String)` creates and initializes a context for a user without setting it in the thread (`OBContext#createOBContext(String)`).

Used in: [01-architecture](./01-architecture.md#layers), [04-security-and-filtering](./04-security-and-filtering.md#reading-guide)

### User level

The user level is the value that `OBContext#setRole(Role)` takes from the role into the user context (`OBContext#setRole(Role)`). `OBContext#setReadableClients(Role)` limits the [readable clients](#readable-clients) to `{"0"}` when it is `S`, and `OBContext#setWritableOrganizations(Role)` adds `"0"` to the [writable organizations](#writable-organizations) when it contains `S` or `C` and removes `"0"` when it is exactly `O` (`OBContext#setReadableClients(Role)`, `OBContext#setWritableOrganizations(Role)`).

Used in: [04-security-and-filtering](./04-security-and-filtering.md#initialization-order)

### UUID

A UUID is a universally unique identifier: `DalMappingGenerator#generateStandardID` maps the single id property with type `string` and `unsaved-value="null"`, and adds the generator `DalUUIDGenerator` (boundary) when the id is a UUID id and is not based on another property (`DalMappingGenerator#generateStandardID`). An id based on another property gets a `foreign` generator instead, and any other id gets no generator element (`DalMappingGenerator#generateStandardID`).

Used in: [01-architecture](./01-architecture.md#per-entity-rules)

### View entity

A view entity is an entity whose AD table is flagged as a view: `Entity#initialize(Table)` copies the flag, which `Entity#isView` reads, and makes the entity not mutable (`Entity#initialize(Table)`). `OBDal#save(Object)` logs a warning and returns without saving a `BaseOBObject` of a view entity (`OBDal#save(Object)`). In the mapping, the properties of a view entity get `update="false"` and `insert="false"`, and a [bag](#bag) whose owning or target entity is a view entity is `mutable="false"` and has no cascade (`DalMappingGenerator#generatePrimitiveMapping`, `DalMappingGenerator#generateOneToMany`). `ModelProvider#initialize` does not take the mandatory flags of a view entity from the database (`ModelProvider#initialize`).

Used in: [01-architecture](./01-architecture.md#per-entity-rules), [02-runtime-model](./02-runtime-model.md#how-the-runtime-model-is-built), [03-dal-service-api](./03-dal-service-api.md#obdalsaveobject)

### Virtual entity

A virtual entity is the second entity that `ModelProvider#initialize` adds, from the same table, for an entity with [computed columns](#computed-column); its name and class name end in `_ComputedColumns` and its table id ends in `_CC` (`ModelProvider#initialize`, `Entity#initializeComputedColumns`). `Entity#isVirtualEntity` marks it, and `Entity#initializeComputedColumns` makes it not deletable, not mutable and inactive, holding the key columns, the computed columns and the `AD_Client_ID` and `AD_Org_ID` columns; its Javadoc says it is mapped to the same database table as the main entity (`Entity#initializeComputedColumns`, `Entity#isVirtualEntity`). `GenerateEntitiesTask#execute` writes no main class for it and `DalMappingGenerator#generateMapping()` skips it, while for the main entity `DalMappingGenerator#generateComputedColumnsClassMapping` appends a separate class mapping named `<package>.<SimpleClassName>_ComputedColumns` on the same table (`GenerateEntitiesTask#execute`, `DalMappingGenerator#generateMapping()`, `DalMappingGenerator#generateComputedColumnsClassMapping`). The [cross-organization reference check](#cross-organization-reference-check) skips references to a virtual entity (`OBInterceptor#checkReferencedOrganizations`).

Used in: [01-architecture](./01-architecture.md#which-entities-are-mapped), [02-runtime-model](./02-runtime-model.md#how-the-runtime-model-is-built), [04-security-and-filtering](./04-security-and-filtering.md#cross-organization-reference-check)

### Writable organizations

The writable organizations are the organization ids to which `SecurityChecker#checkWriteAccess` restricts the organization of an object that the current context writes, apart from three exceptions (`SecurityChecker#checkWriteAccess`). `OBContext#setWritableOrganizations(Role)` adds `"0"` when the [user level](#user-level) contains `S` or `C`, adds the active organizations of the role, removes `"0"` when the user level is exactly `O`, and then adds every organization registered through `OBContext#addWritableOrganization(String)` (`OBContext#setWritableOrganizations(Role)`). When the organization resolved by `OBContext#initialize(String, String, String, String, String, String)` is not among them, it logs a warning and, if any organization is writable, replaces it with the first element that the iterator of the set returns (`OBContext#initialize(String, String, String, String, String, String)`). `OBContext#getWritableOrganizations()` computes the set when none is cached and returns a copy (`OBContext#getWritableOrganizations()`).

Used in: [04-security-and-filtering](./04-security-and-filtering.md#reading-guide)
