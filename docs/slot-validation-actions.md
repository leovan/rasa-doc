# 槽验证动作

槽验证动作是一种特殊类型的自定义动作，旨在处理槽值的自定义提取和/或验证。这可用于验证具有预定义映射的槽或提取具有自定义映射的槽。

你可以通过三种不同的方式编写自定义动作来自定义提取和验证槽值。

## `action_validate_slot_mappings` {#action_validate_slot_mappings}

你可以使用 `action_validate_slot_mappings` 动作来定义可以在表单上下文之外设置或更新的槽的自定提取和/或验证。

此动作在默认动作 [`action_extract_slots`](default-actions.md#action_extract_slots) 结束时自动调用，因此不得更改名称。如果使用 Rasa SDK，你应该扩展 [`ValidationAction` 类](action-server/validation-action.md#how-to-subclass-validationaction)。如果使用不同的动作服务，你将需要实现与 Rasa SDK 类具有等效功能的动作类。有关详细信息，请参阅[动作服务文档](action-server/validation-action.md#validationaction-class-implementation)。

使用此选项，无需在[自定义槽映射](domain.md#custom-slot-mappings)中指定 `action` 键，因为默认动作 [`action_extract_slots`](default-actions.md#action_extract_slots) 会自动运行 `action_validate_slot_mappings`（如果领域的 `actions` 部分中存在）。

## `validate_<form name>` {#validate_form-name}

如果在其名称中指定的表单被激活，名为 `validate_<form name>` 的自定义动作将自动运行。如果使用 Rasa SDK，自定义动作应继承自 [`FormValidationAction` 类](action-server/validation-action.md#formvalidationaction-class)。如果不使用 Rasa SDK，则需要在自定义动作服务中实现具有与 `FormValidationAction` 等效功能的动作或动作类。有关详细信息，请参阅[动作服务文档](action-server/validation-action.md#validationaction-class-implementation)。

## 常规自定义动作 {#regular-custom-action}

你可以使用常规自定义动作[自定义动作](custom-actions.md)，该动作返回用于自定义槽提取的 [`slot`](action-server/events.md#slot) 事件。如果 [`action_validate_slot_mappings`](slot-validation-actions.md#action_validate_slot_mappings) 或 [`validate_<form name>`](slot-validation-actions.md#validate_form-name) 都不能满足需要，请使用此选项。例如，如果你想在故事或规则中明确重用相同的自定义动作，你应该使用常规自定义动作来提取自定义槽。槽验证动作应仅返回 `slot` 和 [`bot`](action-server/events.md#bot) 事件。默认动作 [`action_extract_slots`](default-actions.md#action_extract_slots) 将过滤掉任何其他事件类型。自定义动作的名称必须在领域中相关[自定义槽映射](domain.md#custom-slot-mappings)的 `actions` 键中指定。请注意，动作名称也必须列在领域 `actions` 部分中。

!!! info "使用不同的动作进行提取和验证"

    你可以同时使用常规自定义动作和 `action_validate_slot_mappings` 来提取和验证槽。例如，你可以将常规自定义动作指定为自定义槽映射的动作，并将同一槽的验证逻辑添加到 `action_validate_slot_mappings`。自定义槽映射中指定的自定义动作将首先调用，然后调用 `action_validate_slot_mappings`。
