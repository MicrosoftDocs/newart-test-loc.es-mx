---
title: Visual Filters API en objetos visuales de Power BI
description: En este artículo se explica cómo los objetos visuales de Power BI pueden filtrar otros objetos visuales.
author: KesemSharabi
ms.author: kesharab
manager: rkarlin
ms.reviewer: sranins
ms.service: powerbi
ms.subservice: powerbi-custom-visuals
ms.topic: conceptual
ms.date: 06/18/2019
ms.openlocfilehash: 98ebc87cf5a6b7bf8f0b8b88d4ff498edfd5bf9a
ms.sourcegitcommit: a1c994bc8fa5f072fce13a6f35079e87d45f00f2
ms.translationtype: MT
ms.contentlocale: es-MX
ms.lasthandoff: 10/16/2019
ms.locfileid: "72424548"
---
# <a name="the-visual-filters-api-in-power-bi-visuals"></a>Visual Filters API en objetos visuales de Power BI

Visual Filters API le permite filtrar datos en objetos visuales de Power BI. La principal diferencia con respecto a otras selecciones es que otros objetos visuales se filtrarán de todos modos, a pesar de que otro objeto visual admita el resaltado.

Para habilitar el filtrado del objeto visual, este debe contener un objeto `filter` en la sección `general` del código de *capabilities.json*.

```json
"objects": {
        "general": {
            "displayName": "General",
            "displayNameKey": "formattingGeneral",
            "properties": {
                "filter": {
                    "type": {
                        "filter": true
                    }
                }
            }
        }
    }
```

Las interfaces de Visual Filters API están disponibles en el paquete [powerbi-models](https://www.npmjs.com/package/powerbi-models). El paquete también contiene clases para crear instancias de filtro.

```cmd
npm install powerbi-models --save
```

Si usa una versión antigua (anterior a la 3.x.x) de las herramientas, debe incluir `powerbi-models` en el paquete de objetos visuales. Para obtener más información, vea la guía breve [Adición de Advanced Filter API al objeto visual personalizado](https://github.com/Microsoft/powerbi-visuals-sampleslicer/blob/master/doc/AddingAdvancedFilterAPI.md).

Todos los filtros amplían la interfaz de `IFilter`, como se muestra en el código siguiente:

```typescript
export interface IFilter {
    $schema: string;
    target: IFilterTarget;
}
```
Donde:
* `target` es la columna de la tabla en el origen de datos.

## <a name="the-basic-filter-api"></a>Basic Filter API

En el código siguiente se muestra la interfaz de filtro básica:

```typescript
export interface IBasicFilter extends IFilter {
    operator: BasicFilterOperators;
    values: (string | number | boolean)[];
}
```

Donde:
* `operator` es una enumeración con los valores *In*, *NotIn* y *All*.
* `values` son los valores para la condición.

Ejemplo de filtro básico:

```typescript
let basicFilter = {
    target: {
        column: "Col1"
    },
    operator: "In",
    values: [1,2,3]
}
```

El filtro significa "Enviadme todas las filas donde `col1` es igual al valor 1, 2 o 3".

El equivalente en SQL es el siguiente:

```sql
SELECT * FROM table WHERE col1 IN ( 1 , 2 , 3 )
```

Para crear un filtro, puede usar la clase BasicFilter en `powerbi-models`.

Si usa una versión anterior de la herramienta, debería obtener una instancia de los modelos en el objeto de ventana mediante `window['powerbi-models']`, como se muestra en el código siguiente:

```javascript
let categories: DataViewCategoricalColumn = this.dataView.categorical.categories[0];

let target: IFilterColumnTarget = {
    table: categories.source.queryName.substr(0, categories.source.queryName.indexOf('.')),
    column: categories.source.displayName
};

let values = [ 1, 2, 3 ];

let filter: IBasicFilter = new window['powerbi-models'].BasicFilter(target, "In", values);
```

El objeto visual invoca el filtro mediante el método applyJsonFilter() en la interfaz de host IVisualHost, que se proporciona al objeto visual en el constructor.

```typescript
visualHost.applyJsonFilter(filter, "general", "filter", FilterAction.merge);
```

## <a name="the-advanced-filter-api"></a>Advanced Filter API

[Advanced Filter API](https://github.com/Microsoft/powerbi-models) permite realizar consultas complejas de selección y filtrado de puntos de datos entre objetos visuales que se basan en varios criterios, como *LessThan*, *Contains*, *Is*, *IsBlank*, etc.

El filtro se presentó en Visuals API 1.7.0.

Advanced Filter API también requiere `target` con un nombre de `table` y `column`. Pero los operadores de Advanced Filter API son *And* y *Or*. 

Además, el filtro usa condiciones en lugar de valores con la interfaz:

```typescript
interface IAdvancedFilterCondition {
    value: (string | number | boolean);
    operator: AdvancedFilterConditionOperators;
}
```

Los operadores de condición del parámetro `operator` son *None*, *LessThan*, *LessThanOrEqual*, *GreaterThan*, *GreaterThanOrEqual*, *Contains*, *DoesNotContain*, *StartsWith*, *DoesNotStartWith*, *Is*, *IsNot*, *IsBlank* e "IsNotBlank".

```javascript
let categories: DataViewCategoricalColumn = this.dataView.categorical.categories[0];

let target: IFilterColumnTarget = {
    table: categories.source.queryName.substr(0, categories.source.queryName.indexOf('.')), // table
    column: categories.source.displayName // col1
};

let conditions: IAdvancedFilterCondition[] = [];

conditions.push({
    operator: "LessThan",
    value: 0
});

let filter: IAdvancedFilter = new window['powerbi-models'].AdvancedFilter(target, "And", conditions);

// invoke the filter
visualHost.applyJsonFilter(filter, "general", "filter", FilterAction.merge);
```

El equivalente en SQL es el siguiente:

```sql
SELECT * FROM table WHERE col1 < 0;
```

Para obtener el código de ejemplo completo para usar la Advanced Filter API, vaya al [repositorio del objeto visual Sampleslicer](https://github.com/Microsoft/powerbi-visuals-sampleslicer).

## <a name="the-tuple-filter-api-multi-column-filter"></a>Tuple Filter API (filtro de varias columnas)

Tuple Filter API se presentó en Visuals API 2.3.0. Es similar a la Basic Filter API, pero le permite definir condiciones para varias columnas y tablas.

En el código siguiente se muestra la interfaz de filtro: 

```typescript
interface ITupleFilter extends IFilter {
    $schema: string;
    filterType: FilterType;
    operator: TupleFilterOperators;
    target: ITupleFilterTarget;
    values: TupleValueType[];
}
```

Donde:
* `target` es una matriz de columnas con nombres de tabla:

    ```typescript
    declare type ITupleFilterTarget = IFilterTarget[];
    ```

  El filtro puede abarcar columnas de varias tablas.

* `$schema` es http://powerbi.com/product/schema#tuple.

* `filterType` es *FilterType.Tuple*.

* `operator` permite el uso solo en el operador *In*.

* `values` es una matriz de tuplas de valor y cada tupla representa una combinación permitida de los valores de columna de destino. 

```typescript
declare type TupleValueType = ITupleElementValue[];

interface ITupleElementValue {
    value: PrimitiveValueType
}
```

Ejemplo completo:

```typescript
let target: ITupleFilterTarget = [
    {
        table: "DataTable",
        column: "Team"
    },
    {
        table: "DataTable",
        column: "Value"
    }
];

let values = [
    [
        // the first column combination value (or the column tuple/vector value) that the filter will pass through
        {
            value: "Team1" // the value for the `Team` column of the `DataTable` table
        },
        {
            value: 5 // the value for the `Value` column of the `DataTable` table
        }
    ],
    [
        // the second column combination value (or the column tuple/vector value) that the filter will pass through
        {
            value: "Team2" // the value for `Team` column of `DataTable` table
        },
        {
            value: 6 // the value for `Value` column of `DataTable` table
        }
    ]
];

let filter: ITupleFilter = {
    $schema: "http://powerbi.com/product/schema#tuple",
    filterType: FilterType.Tuple,
    operator: "In",
    target: target,
    values: values
}

// invoke the filter
visualHost.applyJsonFilter(filter, "general", "filter", FilterAction.merge);
```

> [!IMPORTANT]
> El orden de los nombres de columna y los valores de condición es sensible.

El equivalente en SQL es el siguiente:

```sql
SELECT * FROM DataTable WHERE ( Team = "Team1" AND Value = 5 ) OR ( Team = "Team2" AND Value = 6 );
```  

## <a name="restore-the-json-filter-from-the-data-view"></a>Restauración del filtro JSON desde la vista de datos

A partir de la versión 2.2 de la API, puede restaurar el filtro JSON desde *VisualUpdateOptions*, tal y como se muestra en el código siguiente:

```typescript
export interface VisualUpdateOptions extends extensibility.VisualUpdateOptions {
    viewport: IViewport;
    dataViews: DataView[];
    type: VisualUpdateType;
    viewMode?: ViewMode;
    editMode?: EditMode;
    operationKind?: VisualDataChangeOperationKind;
    jsonFilters?: IFilter[];
}
```

Al cambiar los marcadores, Power BI llama al método `update` del objeto visual y este obtiene un objeto `filter` correspondiente. Para obtener más información, consulte [Adición de compatibilidad de los marcadores para los objetos visuales de Power BI](bookmarks-support.md).

### <a name="sample-json-filter"></a>Filtro JSON de ejemplo

En la siguiente imagen se muestra un código de filtro JSON de ejemplo:

![Código de filtro JSON](./media/json-filter.png)

### <a name="clear-the-json-filter"></a>Borrado del filtro JSON

Filter API acepta el valor `null` del filtro como *reset* o *clear*.

```typescript
// invoke the filter
visualHost.applyJsonFilter(null, "general", "filter", FilterAction.merge);
```
