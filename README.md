# block-editer

请帮我使用 ProseKit 来复刻 wordware 的基本功能，如果涉及到请求，用 mock + Promise 的形式来模拟就好。

我希望使用 vue3 来开发，功能都需要，但我希望是扩展的形式完成，不要用硬编码，这样扩展性更好，独立插件实现每一个扩展。

---

Vue3 + ProseKit 实现可插拔 AI 编辑器骨架调研

ProseKit 插件扩展机制概览

ProseKit 基于 ProseMirror，提供了插件化的扩展机制，允许我们通过扩展（extension）模块添加自定义功能，而非在核心代码中硬编码逻辑。每个扩展都是模块化的，可以提供编辑器所需的各种元素，包括：
	•	自定义节点：如段落、标题、列表、代码块等节点类型 ￼。
	•	文本标记：如粗体、斜体、下划线、删除线等行内格式。
	•	命令和快捷键：扩展可以注册编辑命令（如插入某节点、切换格式）及对应的键盘快捷方式。
	•	编辑器插件：监听编辑器状态、光标事件等，实现定制行为。

ProseKit 将这些扩展组合生成最终的编辑器配置。在使用时，我们通常创建一个 Editor 实例并传入需要的扩展。例如 ProseKit 提供了 defineBasicExtension() 来创建基础富文本扩展，它包含段落、标题、粗体、斜体等常见功能 ￼。可以将多个扩展通过 union(...) 等方法合并注册，然后传递给 createEditor 来实例化编辑器。ProseKit 会自动合并各扩展的 Schema 定义和插件。例如，其 Changelog 提到：可以对同一节点名称多次调用 defineNodeSpec/defineMarkSpec，它们的 spec 会被自动合并，方便多个插件向同一节点或标记添加属性或行为 ￼。这种机制保证了插件的解耦和协作，让我们可以自由添加或移除功能模块而不会冲突。

在 Vue3 集成中，ProseKit 提供了对应的 Vue 支持包（prosekit/vue）。它包含一个Editor根组件（例如 <ProseKit>）用于挂载编辑器，以及 Composition API 方法用于在 Vue 组件中注册扩展、获取编辑器实例等 ￼。通常我们会：
	1.	使用 createEditor 创建编辑器实例（可传入扩展列表、初始内容等配置）。
	2.	将编辑器实例绑定到 <ProseKit> 组件，使其渲染实际的富文本区域。
	3.	在各功能组件中，通过 ProseKit 提供的 useExtension 等方法向编辑器注册各插件扩展，并使用 provide/inject 获取编辑器上下文，实现插件的按需加载和模块化注册。

总之，ProseKit 的扩展体系使我们能够以模块化、可插拔的方式构建编辑器，将不同功能封装为独立插件，避免硬编码在一个庞大的代码基中。

“/” 命令插入（Slash Command）插件设计

Slash 命令（斜杠命令）插件允许用户在编辑器中键入 / 触发一个弹出菜单，从中选择插入各种模块或节点。要实现这一功能，我们可以利用 ProseKit 的自动完成（Autocomplete）扩展机制。具体思路如下：
	•	触发检测：扩展需要监听用户输入，当检测到在特定条件下输入了/时激活菜单。通常要求/位于段落开头或紧跟空格后 ￼（避免干扰普通文本输入）。ProseKit 提供了 defineAutocomplete 等 API 来定义触发规则（例如触发字符 / 及后续输入的匹配模式），插件据此判断何时显示候选列表。
	•	菜单项及过滤：Slash 插件维护一组“命令”项供选择。每个项代表可插入的模块（如文本、标题、If-Else节点等），包含显示名称、描述和执行动作。这些项 ideally 由各模块插件注册提供，以避免由 Slash 插件硬编码。实现上可以让 Slash 插件接受一个菜单项数组参数，初始化时合并所有模块的项。这样新增模块时，只需将其命令项加入数组即可。插件激活时，会根据用户在斜杠后的输入对项列表进行过滤，仅显示匹配的项 ￼。用户可以通过键盘上下键导航选项，并实时筛选列表（输入字符会继续插入到编辑器暂存，插件根据编辑器内容过滤） ￼。
	•	异步搜索支持：如果某些命令项需要异步获取（例如搜索数据库中的某些内容插入结果），可以将该项的提供方式设计为异步回调。当输入匹配该项时，先显示“加载中”状态，然后调用异步函数获取结果并更新菜单列表。这要求 Slash 插件能动态更新其建议项。ProseKit/ProseMirror 插件可以通过更新自身的 state 或触发重新计算建议来实现这一点。社区的一些实现（如 Emergence Engineering 的 prosemirror-slash-menu 插件）就是将 Slash 插件拆分成纯状态逻辑和UI 展示两部分：ProseMirror 插件仅处理输入检测、建议项过滤、选中项索引等状态，而实际下拉菜单 UI 由框架组件实现，通过读取插件提供的数据来渲染 ￼。这种分离符合我们在 Vue 中的设计——逻辑在插件，UI 在组件。
	•	UI 展示与交互：在 Vue3 中，我们可以创建一个 SlashMenu 组件用于渲染下拉菜单。该组件通过 inject 获取编辑器实例或通过 ProseKit 提供的 Composition API 获取 Slash 插件的状态（如当前候选项列表、选中项索引、菜单锚点位置等）。Slash 插件可以通过在文档中插入一个装饰（decoration）或利用 ProseMirror 提供的坐标计算，在触发位置放置一个虚拟的锚点元素，用于确定弹窗的定位 ￼。Vue 组件拿到锚点位置后，可使用绝对定位将菜单浮层显示在光标附近。用户按上下键时，插件拦截按键事件更新选中索引（避免光标在文档中移动） ￼，组件则根据索引高亮对应项。用户按 Enter 确认选择时，插件应阻止回车插入换行，改为执行所选命令。这可以通过插件监听 Enter 键在菜单打开状态时的行为实现。用户点击菜单项时，我们可以通过 Vue 组件调用编辑器命令的方式触发同样的插入逻辑。
	•	执行插入：当用户选择了某个命令项（无论键盘或点击），Slash 插件需要插入对应模块。最佳实践是由各模块插件提前定义好插入该模块的命令或方法，例如 IfElse 插件定义 insertIfElse 命令，变量插件定义 insertVariable 命令等。Slash 插件通过调用这些命令来完成插入，这样新增模块时只需提供自己的插入逻辑，Slash 插件本身不需要修改。命令调用可以借助编辑器实例提供的 API（类似 Tiptap 的 editor.commands.<command>() 调用链）。在 ProseKit 中，扩展可以通过定义命令扩展（如 defineMentionCommands 等）向编辑器注册命令 ￼，因此我们可以让每个模块插件在其扩展中注册一个插入命令函数，然后在 Slash 菜单项中引用该命令。

综上，Slash 命令插件的实现步骤为：
	1.	定义触发规则：使用 defineAutocomplete 或自定义 ProseMirror 插件，在检测到 / 时激活建议模式。
	2.	准备菜单数据：收集所有可用的命令项（名称、图标、描述、执行函数），支持异步获取和动态过滤。
	3.	维护插件状态：跟踪当前输入的过滤关键字、候选列表、选中索引，拦截相关按键（上下/回车/ESC）。
	4.	渲染 Vue 组件：通过 Editor 提供的状态定位弹窗，并将候选项和交互逻辑用 Vue 展现出来。
	5.	调用插入动作：将最终选择映射为编辑器命令，插入相应内容。比如用户选择插入 If-Else 模块，相当于执行 IfElse 扩展提供的插入命令，在文档中插入一个 If-Else 节点。

参考实践：已有的开源实现如 prosemirror-slash-menu 就采用了类似架构：/ 打开菜单、菜单项通过数组配置、支持键盘导航和嵌套子菜单 ￼。在 Tiptap 中集成该插件时，也是创建一个扩展包装 ProseMirror 插件 ￼。这些都印证了 Slash 插件宜作为独立扩展加入编辑器，并通过灵活配置和UI组件实现高度可扩展的命令菜单。

值得一提的是，Wordware 就是通过斜杠命令提供各种节点插入功能的典型例子。例如，在 Wordware 编辑器中输入 “/if” 然后回车，即可插入一个 If-Else 节点，并弹出该节点的属性配置界面供填写细节 ￼。我们在复刻时也会采用类似交互：所有逻辑块的创建都通过 / 菜单完成，这样保持一致的用户体验和可扩展性。

“@” 引用（Mention）插件设计

@引用（Mention）插件允许用户输入@来引用当前上下文中的变量、模块输出或其他逻辑块。其行为类似于常见的@提及功能：输入@后出现可选项列表，选择后在文档中插入一个特殊引用节点。实现上，可以复用与 Slash 命令类似的建议/自动完成机制，但数据来源和插入内容有所不同：
	•	触发与候选获取：当用户在编辑器中键入@时，Mention 扩展应检测并打开建议列表 ￼。初始列表包含当前上下文中“所有可引用项” ￼。根据 Wordware 文档，引用项可以包括之前生成的内容块、用户输入（Input节点）、其他 Flow 的输出，甚至代码执行的结果 ￼——总之就是在编辑器或应用范围内已有的命名内容。我们需要确定这些可引用项的来源：
	•	可以维护一个全局 registry，在每次新建命名模块（如 Input、变量声明、Flow 调用等）时登记其名称和引用值。
	•	或者在打开@建议时动态遍历当前文档，搜集所有有标识符的节点（比如有 name 属性的节点）。
	•	也可以允许通过 API 提供外部数据（比如当前上下文中传入的变量集合）。
最简单做法是插件每次触发时扫描文档节点，过滤出可引用对象并生成列表。对于大量节点这可能需要优化，因此更好的方案是在插件状态中维护索引：每当文档有变动（transaction）时，插件检查新增或删除的节点，如果它包含可引用名称，则更新内部列表。这样在按下@时，已有一个最新的可引用项集合，可以按名称或类型分类展示。
	•	建议列表交互：与 Slash 类似，用户可以继续输入过滤。比如输入@us可能筛选出名为“userName”的变量或“User Input”等。插件需要实时根据@后字符更新列表 ￼。键盘上下选择、Enter确认、ESC取消等交互也类似，可复用自动完成插件通用逻辑。因此，我们可以将 Mention 看作是Autocomplete 插件的一个特例：只不过数据源是当前环境的可引用实体而非固定命令列表（正如 Milkdown 的 Slash 插件也可用于 @ 提及 ￼）。
	•	插入引用节点：当用户选中某个引用项（变量/模块）并回车确定时，插件应在文档中插入一个引用节点。通常这个节点是一个行内（inline）节点，内容上可能显示为被引用对象的名称或占位符，但不会是普通文本，而是在文档结构中作为特殊节点存在。我们可以利用 ProseKit 提供的 defineMention 或直接用 defineNodeSpec 定义这样的节点：
	•	该节点可以叫mention，有属性例如targetId或targetName标识引用目标，以及targetType（比如引用的是变量或模块类型，以便渲染时区分）。
	•	节点的 schema 可以是不允许再编辑其内容（避免用户破坏引用的一致性），表现为一个原子 inline node。
	•	插入节点后，在内容上可以渲染为 @名称 形式。这个渲染由NodeView或通过自定义 schema 的 toDOM 实现。ProseKit 提供了封装方法定义 NodeView，例如 defineVueNodeView 等，可直接将 Vue 组件作为节点的视图 ￼。我们可以编写一个 Vue 组件 MentionNode 来根据 targetType 渲染不同图标或样式（比如引用变量用一种颜色，引用模块输出用另一种标签）。
	•	引用节点渲染与更新：由于引用节点可能需要显示最新的值（比如引用一个变量的当前值），渲染逻辑需要能获取引用目标的内容。简单起见，可以在引用节点插入时就将引用的内容副本存入节点属性用于显示。但为了保持实时更新（例如变量值在运行过程中变化），更理想的是引用节点只存标识符，显示时通过查找外部数据源获取实际值。在编辑器静态编辑阶段，引用显示的主要是名称/占位符即可，真正执行时再替换为值 ￼。Wordware 就是在运行时（runner）将mention替换为它引用的内容 ￼。
	•	插件状态与 UI：Mention 插件在原理上和 Slash 类似，都需要监听编辑器变化来决定何时弹出建议。当@输入后，插件可以插入一个inline decoration（零宽度）在@字符位置，并在该位置附加一个带唯一ID的 DOM 元素，以便由外部UI（Vue组件）锚定 popup ￼。这个装饰还可以带特殊样式让@后的临时文本高亮，提示用户正在输入引用。插件应追踪用户从@开始输入的字符串，每当编辑器 state 更新时检查光标是否紧跟在@mention装饰后并获取最新文本 ￼。当用户按下空格、标点或 ESC 等导致提及中止时，插件应取消建议状态并清除临时文本装饰。如果用户确认选择，则执行插入节点的事务并结束建议。这个过程在实现上通常通过 ProseMirror 插件的 apply(tr, oldState, newState) 来管理内部状态机 ￼。

Mentions 实现要点：
	1.	节点定义：定义 mention 节点 schema 和 NodeView，使其在文档中作为不可编辑的原子单元显示引用 ￼。
	2.	数据提供：确定可引用项列表的维护方式，建议在插件 state 中维护，并提供按输入过滤的方法。
	3.	触发逻辑：监听输入@开启建议模式，通过 decorations 定位弹窗并保留正在输入的文本 ￼。
	4.	组件显示：类似 SlashMenu，实现一个 MentionSuggestions 下拉组件，读取插件提供的候选项及当前位置，在 UI 上展示匹配列表。根据 Wordware 文档，应当显示所有可用mentions并支持搜索过滤 ￼。
	5.	选择插入：当选定某项时，插件插入 mention 节点，节点属性带上目标标识。可通过定义命令（如insertMention(target)）来完成，以便将来别的地方（比如点击 UI 项）也能调用。
	6.	模块化：Mention 插件本身可以对不同类型的引用项保持通用，但如果某些类型引用需要特别处理，可以进一步拆分。例如，可以有专门的 Variable 插件提供变量类型数据，Flow 插件提供 Flow 输出数据，Mention 插件汇总它们。不过如果实现复杂，可以初期让 Mention 插件直接知晓所有类型来源，后续再拆分优化。

通过这样的设计，@引用功能将与 Wordware 的行为一致：用户在编辑器任意位置输入@，即弹出列表供选择要嵌入的内容引用 ￼。插入后，这些引用节点在编辑阶段充当占位标记，提示所引用的内容名称；在实际运行（例如执行 prompt 时），系统会将其替换为对应的实际内容 ￼。这种插件化实现确保了引用功能与其他模块解耦。例如，将来如果我们增加新的可引用实体类型，只要将其纳入 Mention 插件的数据来源即可，核心逻辑无需修改。

富文本基础编辑扩展

一个 AI 编辑器的基础是丰富的文本编辑能力，包括段落、标题、列表、代码块、引用等基本格式。ProseKit 已经提供了现成的基础扩展来实现这些功能：
	•	基础扩展 Basic：defineBasicExtension() 一键启用常见的块和行内格式，例如段落、各级标题（Heading）、加粗、斜体等 ￼。使用该扩展可以快速获得大部分富文本能力，而无需自行定义所有 schema 和命令。Basic 扩展内部可能集成了一些子扩展或规则，例如列表、引用等。特别地，ProseKit 采用了 prosemirror-flat-list 实现列表节点 ￼。Flat List 能避免 ProseMirror 默认嵌套 list 难以管理的问题，用一种扁平结构表示有序/无序/任务列表，提高了稳定性和可控性。通过 Basic 扩展或相应的 List 扩展，我们可以获得无嵌套列表支持，用户可直接在编辑器中输入 - 、*  或数字+. 来触发列表项，列表项缩进、出坑等操作由扩展内部键盘绑定处理（例如 Tab/Shift+Tab 改变层级）。
	•	代码块与高亮：ProseKit 支持多行代码块节点，以及代码高亮功能。高亮通常借助第三方库，如 prosemirror-highlight 提供的解析器。ProseKit 有对应的代码块扩展，可以通过传入一个 syntax parser 来给代码块内容着色 ￼。实现时，我们可使用 defineCodeBlockExtension（假设存在）并提供一个高亮解析器（例如 Prism.js 的 ProseMirror 适配），从而在编辑器中实时呈现代码高亮。需要注意高亮通常通过装饰实现，不会改变文档内容，只是为代码文本添加带样式的<span>。这个扩展同样是插件化的，可根据需要启用或禁用。
	•	引用（Blockquote）：引用块通常作为独立节点类型，可能已包含在 Basic 扩展中。如果没有，可以使用 defineNodeSpec('blockquote', ...) 定义一个具有专有样式的容器节点，并附带输入规则（例如在行首输入> 触发引用块）。ProseKit Basic 很可能已经集成了这种输入规则。
	•	Inline 格式：如代码（行内）、删除线等。如 Basic 扩展未涵盖全部，可以用 defineMarkSpec('code', ...) 等添加行内代码标记，以及对应的快捷键/输入规则（例如`content`包裹触发）。扩展机制允许多次定义 mark spec 并合并 ￼，因此即使 Basic 提供了一部分，我们仍可通过额外扩展添加新的行内格式支持而不冲突。

在实际项目中，我们可以采取按需组合的方式构建基础编辑功能：
	•	启用 Basic：快速获得段落、基本文本格式和可能的列表/引用支持。
	•	补充扩展：如果 Basic 未包含列表或表格等，我们引入 ProseKit 对应扩展。例如 List 扩展（基于 flat-list）保证列表行为良好；Table 扩展（如果有）提供表格节点；Task list 扩展提供可勾选的任务列表项等等。
	•	代码块扩展：启用代码块+高亮，让编辑器支持插入代码段并着色 ￼。
	•	历史和撤销：不要忘记基础的撤销/重做。ProseKit 可能有 defineHistory() 扩展来注入 ProseMirror 历史插件（撤销栈），以便 Ctrl+Z/Y 可用。
	•	剪贴板和其他：有些额外细节如粘贴HTML、拖拽图片等，也可通过插件扩展实现（如果需要，可以寻找 ProseMirror 已有插件或 ProseKit 扩展）。

通过模块化启用以上扩展，我们无需重新发明轮子就能获得与典型富文本编辑器相当的基础能力。重要的是，这些基础功能作为独立扩展存在，可以和我们自定义的 AI 节点（如逻辑块）并存，而不会相互干扰。例如，Basic 扩展提供的节点（段落、标题等）会并入编辑器 schema，我们自定义的节点（IfElse、Loop 等）也会加入；ProseKit 会将它们合并成统一的 schema 和命令集合。因此，在最终编辑器中，用户既可以进行常规的文本文字编辑和格式调整，又可以通过我们添加的特殊节点来构建逻辑。

逻辑节点（If/Else、循环等）的扩展实现

Wordware 的核心亮点在于逻辑节点系统，比如条件判断 (If-Else)、循环 (Loop)、变量声明、函数调用等。这些节点并不像普通文本那样简单，而是具有内部结构和业务逻辑。在 ProseKit 中，我们可以将每种逻辑功能封装为一个Block/Node 扩展，使其成为文档中的自定义节点类型。下面以 If-Else 节点为例，说明如何设计这些扩展：
	•	Schema 定义：使用 ProseKit/ProseMirror 的 NodeSpec 定义新节点。以 If-Else 为例，我们需要表示一个条件判断结构，可能包含若干条件分支及对应内容。可以设计成：
	•	一个父节点类型，例如 if_block，作为整个 If-Else 容器。
	•	子节点类型：if_condition 和 else_condition（或者用属性区分不同分支）。每个条件节点包含一个表达式和一个内容块。
	•	Schema 可以规定 if_block 的内容模型，例如：必须包含一个或多个 if_condition 节点，最后可以可选一个 else_condition 节点。
	•	每个条件节点内部再允许嵌入任意块节点作为条件成立时要执行的“动作”（子内容）。
	•	此外，还可以给这些节点设置属性，例如 name（节点名称，用于引用或标识）、每个条件的表达式文本等。
通过这样的 Schema，我们能在文档树中完整表示嵌套的逻辑关系。值得注意的是，ProseMirror 默认的 schema 设计并不直接支持“一节点多槽位”的结构，但我们可以通过层级嵌套来表现（if_block 下多个条件节点，每个条件节点内部一个内容 fragment）。扩展可以借助 ProseKit 的 schema 合并特性，将这些定义注入编辑器 ￼。
	•	节点交互行为：定义 If-Else 时，要考虑用户如何在编辑器中操作它。例如：
	•	当光标在 If-Else 节点内时，按 Enter 可能需要特殊处理：在条件块内回车也许应该在同一条件下创建新段落，而不是跳出节点。
	•	当用户通过 Slash 菜单插入 If-Else 节点后，我们可以自动插入一个默认结构（比如一个初始 IF 条件和对应占位内容，并附加一个可添加新条件的占位）。这样用户看到的是一个 If-Else 模板，接下来可以编辑条件表达式、填充分支内容。
	•	可以提供键盘快捷操作，比如在条件块结尾按下特定组合键（或通过 slash 命令）添加新的 “Else If” 分支节点。
	•	这些行为可以通过扩展的命令或键映射实现。例如扩展可以定义 /else 命令在当前 if_block 后追加 else 分支，也可以监听 Enter 键根据上下文决定行为。
	•	NodeView 定制渲染：逻辑节点往往需要特殊的呈现，比如边框、背景、高亮标题或图标，以及可能需要禁止用户直接编辑某些部分（如条件表达式可能用单独的输入框）。ProseMirror 提供 NodeView 接口允许我们接管节点的渲染和交互。ProseKit 为 Vue3 提供了便捷的方法 defineVueNodeView 来将 Vue 组件绑定为 NodeView ￼。我们可以为每种逻辑节点创建对应的 Vue 组件，例如：
	•	<IfBlockView>：渲染 If-Else 容器。它内部可能循环渲染其子条件节点的视图，并提供整体的外框、标题（比如显示节点名称）以及一个在末尾添加Else分支的按钮。
	•	<IfConditionView>：渲染单个 IF 或 ELSE IF 条件分支。里面包含一个表达式编辑区域和一个子节点内容区域。表达式编辑我们可以用一个 input 或 contenteditable 实现，但要注意同步回写到节点属性。ProseKit 的 MarkView/NodeView 或直接使用 contenteditable div都可以实现双向绑定，或更简单可以让表达式作为节点文本内容（不过那会混入doc，可能不理想）。更好的方式是 NodeView 内部管理一个独立的输入控件，将其值通过更新 ProseMirror transaction 写入节点属性。
	•	<ElseConditionView>：类似 IfConditionView。
通过 NodeView，我们可以实现复杂的交互界面，例如条件的展开折叠、拖拽排序、删除按钮等，完全由Vue组件掌控。同时，因为 NodeView 将节点渲染隔离成独立DOM，用户在组件内的操作（比如改条件文本）我们可以拦截后通过编辑器命令应用，从而避免直接修改文档导致不一致。
	•	命令与事件：每个逻辑节点插件应定义一些命令API，方便外部（包括 Slash 菜单、工具栏按钮等）调用以操作该节点。例如：
	•	insertIfBlock(name?): 在当前光标处插入一个新的 if_block 节点，附带默认空白结构和可编辑名。
	•	addElseCondition(ifBlockId): 给指定 if_block 添加一个 else 分支（或 else if）。
	•	toggleCondition(ifBlockId): 切换条件节点的收起/展开状态（如果实现折叠）。
	•	等等，根据需要提供。
这些命令封装在扩展中，注册到编辑器的 commands 列表中。这样 Slash 插件或其他UI只需简单调用，如 editor.commands.insertIfBlock() 就能执行相应动作 ￼（类似Tiptap的chain调用机制）。另外，可以利用 ProseKit 的事件系统（若有）或者直接利用Vue事件：例如 IfBlockView 组件内“Add Else”按钮点击时，通过触发一个自定义事件或者直接调用注入的editor实例方法来执行命令。
	•	示例：Wordware 中 If-Else 节点在插入后，会弹出一个 Attributes 侧边栏要求填写名称、条件表达式等 ￼。我们在实现中可以选择行内编辑或弹出属性面板两种方式。行内编辑通过 NodeView 已能实现（在编辑器中直接编辑表达式文本区域、名称等）。如果想模仿 Wordware 的侧栏编辑体验，可以在节点插入后触发一个 Vue 事件，在编辑器旁边渲染一个属性编辑面板组件，表单双绑到节点属性。不过在深度集成的编辑器里，尽量让用户直接在节点视图中修改属性会更直观一些。不管采用哪种，关键是逻辑节点要将结构与属性清晰地体现在 schema 和 NodeView 中，并提供良好的交互。

除了 If-Else，其他逻辑节点（Loop循环、Input输入、Tool函数调用等）可以用类似思路各自实现扩展：
	•	Loop：定义 loop_block 节点，包含循环条件和循环体子节点。提供 NodeView 显示循环标识以及迭代次数设置等。Slash 命令 /loop 插入 ￼。
	•	Input：定义 input_node，可能只是内联节点或块，其值可以编辑或者从外部注入。NodeView 可渲染一个占位符框，提示这是一个输入。将 input 插入文档允许后续 mention 引用它的值 ￼。
	•	Comment：Wordware 还有注释节点，逻辑上编辑时可见、AI执行时忽略，这可以定义 comment_block 节点并通过 NodeView使其呈现为灰色框且不可被AI看到（可在生成提示时过滤掉这些节点内容）。
	•	Flow调用：可以有节点表示调用另一个 Flow，或执行某段代码，这些都可以作为节点，通过属性配置调用目标或代码片段，执行时替换为结果。

通过将每个逻辑单元设计为自包含的插件扩展，我们实现了高度可扩展的逻辑系统。将来增加新的逻辑类型（比如一个自定义过滤节点）也只需另写一个扩展，不影响现有部分。这些节点在编辑器 schema 中可能彼此独立，但通过我们构建的 Mention 引用和运行时机制，它们之间形成联系（例如 Input 节点的输出可被 Mention 节点引用到 Generation 节点的提示中）。这种架构既保证了编辑体验的一致性，又方便功能扩充，真正达到 Wordware 那样的插件驱动AI编辑器框架。

插件组织方式和注册机制

在实现以上功能时，合理组织插件代码和注册方式至关重要。基于之前分析，我们推荐如下的插件组织策略：
	•	按功能模块分包：将每类功能放入独立的模块目录。例如：
	•	/extensions/basic：封装富文本基础扩展相关的配置（其实可直接使用 ProseKit Basic，但如果需要定制也可在此扩展或包装）。
	•	/extensions/slash：Slash 命令扩展（包括ProseMirror插件逻辑和Vue组件）。
	•	/extensions/mention：@引用扩展。
	•	/extensions/logic/if-else、/extensions/logic/loop 等：每种逻辑节点一个子模块，内部包含schema定义、命令、Vue视图等。
模块内可以按照ProseKit的概念拆分文件，例如 extension.ts 定义扩展(main入口，导出一个扩展对象或工厂函数)、nodeview.vue 定义 NodeView 组件、commands.ts 定义命令等。
	•	插件注册中心：在应用启动或编辑器创建时，有一个统一地方收集并注册所有需要的扩展。例如创建编辑器前：

import { createEditor } from 'prosekit/core';
import { union } from 'prosekit/core';
import { basicExtension } from './extensions/basic';
import { slashExtension } from './extensions/slash';
import { mentionExtension } from './extensions/mention';
import { ifElseExtension } from './extensions/logic/if-else';
import { loopExtension } from './extensions/logic/loop';
// ... import other extensions as needed

const editor = createEditor({
  extension: union(
    basicExtension(),
    slashExtension(),
    mentionExtension(),
    ifElseExtension(),
    loopExtension(),
    // ...其他扩展
  ),
  content: '<p></p>' // 初始内容
});

这里我们利用 union 将多个扩展合并。【注: ProseKit支持将多个扩展组合，这里假设basicExtension等返回的是可传入createEditor的扩展对象】。这种集中注册方式确保编辑器启动时加载所有所需功能。如果需要按需加载/卸载插件（动态插拔），也可以保留对editor实例的引用，稍后通过 editor.use(extension) 或 ProseKit 提供的上下文方法来添加（视ProseKit API而定）。不过一般来说，大部分编辑功能在初始化时就确定加载。

	•	避免硬编码耦合：各插件模块应通过扩展API通信，而非直接相互调用内部函数。例如，Mention 插件不直接依赖具体的 Input 或 Flow 插件实现，而是通过统一的方式获取可引用项列表。可以设计一个协议：
	•	例如，约定每个可引用节点在schema定义时都包含属性 { name: string } 作为标识。
	•	Mention 插件在扫描文档时，只认节点有 name 属性者，将其 name 收集为可用引用项。
	•	这样Input节点、Flow调用节点等只要有name属性，自动被mention纳入可引用范围，无需进一步适配。
又如 Slash 插件，不依赖具体逻辑插件细节，而是通过传入的菜单项数组驱动。各逻辑插件导出自己的菜单项配置（可以在各自extension.ts中定义好label和执行命令），Slash 插件统一汇总。 ￼ ￼
	•	扩展交互：某些插件可能需要协调。例如插入If-Else节点后，应自动触发在属性面板显示其可编辑属性；或者当删除一个变量节点时，Mention插件需要更新列表。为此，可以利用ProseMirror PluginState或事件总线：
	•	ProseMirror 插件可以定义 filterTransaction 或在 apply 时检查特定节点是否被删，若是则触发自身状态更新。比如 Mention 插件看到某 transaction 从文档中移除了 name=“X” 的节点，则从建议列表中移除X。
	•	Vue3 提供provide/inject和emit机制。我们可以在 Editor 提供一个 EventEmitter或简单的mitt实例，各插件组件监听或触发相应事件。例如 IfElseView 组件在节点插入完成后，通过 $emit('nodeInserted', {type: 'ifElse', node}) 通知父级；Mention 插件（如果以组件形式存在或通过全局事件监听）捕获到此事件，调用内部方法刷新引用源。根据项目复杂度，选择是否引入状态管理库协调：如果逻辑较多，可以引入 Pinia，在其 store 中定义例如 state.references = [] 列表，Input/Flow插件在创建新变量时向该列表push，Mention插件只需引用此 Pinia 状态并过滤显示。这也不失为一种模块解耦手段，尤其当引用源不仅来自编辑器文档也可能来自外部输入时，中央状态库更合适。
	•	命名规范：确保各扩展的命名不冲突，例如节点类型命名使用带前缀的字符串（如 "if_block" 而不是 "if"，防止与内置Schema或其他插件冲突），命令名称也是如此。同时，通过配置优先级（priority）来调整扩展应用顺序，如基础扩展应先于自定义节点加载，以免基础段落规则影响我们特殊块节点的处理 ￼。
	•	开源参考：可以参考 Tiptap 等编辑器的插件组织方式，每个扩展一个类/文件，最终在Editor实例化时组合 ￼。ProseKit 本身的源码和示例（如 image 插件定义 NodeView ￼、Yjs 协同扩展等）也提供了组织插件的范例。利用这些经验我们能够规划清晰的目录结构和依赖关系，使项目易于维护和扩展。

综言之，模块清晰、接口契约明确是插件组织的关键。在我们的设计中，每个功能插件皆可独立存在，通过编辑器将它们串联起来协同工作。这正符合 Wordware “所见即所得编排 + 模块化功能”的思想，插件架构也方便未来对标增加如触发器（Triggers）、工具（Tools）等新节点类型——一切皆插件，无需改动编辑器核心。

Vue3 集成中的组件与状态管理

将 ProseKit 与 Vue3 结合，需要处理好组件注册和状态联动问题。所幸 ProseKit 已提供 Vue 支持，使我们能够优雅地将编辑器嵌入 Vue 应用。以下是集成要点和建议：
	•	Editor 容器组件：使用 ProseKit 提供的 <ProseKit>（或类似）组件作为编辑器挂载点。 ￼示例暗示了用 <ProseKit> 包裹编辑器，以及我们在外部通过 createEditor({...}) 创建编辑器实例并传递给它。这个组件内部会处理将 ProseMirror EditorView 挂载到 DOM、应用我们提供的扩展和内容等繁琐过程。我们可以在主页面的模板中这样使用：

<ProseKit :editor="editorInstance">
  <!-- 可以有默认插入的内容或子组件 -->
</ProseKit>

在中先调用 const editorInstance = createEditor({ extension, content }) 得到实例。ProseKit 组件会通过 Vue Provide 将编辑器上下文传递给子组件（如菜单组件）使用 ￼。注意确保 <ProseKit> 组件的样式（高度、边框等）适当设置，例如占满父容器等。

	•	功能子组件：Slash 菜单、Mention 建议框、各逻辑节点的 NodeView 组件，都是编辑器的子组件，需要访问编辑器状态或触发编辑操作。借助 Provide/Inject，这些组件可以通过 inject 获取到 Editor 对象或者特定的扩展API。例如 ProseKit 可能提供 useExtension(extension) 方法，将扩展和组件关联。如果我们将 Slash 插件的 ProseMirror插件部分在 /extensions/slash/extension.ts 定义好了，也许 ProseKit 有办法让我们在 Vue 组件中启用它 —— 例如在 SlashMenu.vue 中调用：

import { slashExtension } from './extension';
const ext = slashExtension();
useExtension(ext);

这可能会将 Slash 插件注册到当前编辑器实例上（ProseKit 的实现可能是在父 provide 的editor上调用 editor.use()，具体以文档为准）。这样，我们可以选择按需加载插件：仅当渲染了相关组件才注册插件逻辑。这对于 Slash/Mention 这样的浮窗组件是适用的，因为它们只有在需要时才出现。不过在大多数情况下，我们会在初始化就加载所有插件，避免动态添加造成的 schema 变化复杂性。

	•	NodeView 组件注册：对于 NodeView 类型的组件（如 IfBlockView.vue），需要告知编辑器在渲染对应节点时使用它。ProseKit 提供了 defineVueNodeView({ component: YourComponent, as: NodeViewDOMSpec }) 之类的API ￼。一般在扩展定义时使用，比如：

import IfBlockView from './IfBlockView.vue';
const ifNodeSpec = defineNodeSpec('if_block', { /* schema定义 */ });
const ifNodeView = defineVueNodeView({ component: IfBlockView });
export function ifElseExtension() {
  return union(ifNodeSpec, ifNodeView, ...);
}

这样编辑器在遇到if_block节点时，会实例化我们的Vue组件来展示。组件内可以通过inject拿到该节点的内容或属性（可能ProseKit通过props传入）并渲染UI。例如IfBlockView的props可能包含node（PM Node对象）和editor实例引用。有了这些，组件就能根据节点属性显示名称、遍历子节点列表生成子组件等。同时在组件内部，如果需要更新节点（比如用户编辑了条件文本），可以调用提供的editor方法 dispatch一个transaction更新该节点的attrs。

	•	响应式状态管理：Vue3 的优势在于其响应式系统。我们可以监听编辑器状态的变化，以触发Vue组件更新。编辑器本身不是响应式对象，但我们可以利用 ProseMirror 的更新机制，在每次交易后通知Vue重新计算需要的内容。
	•	ProseKit 可能提供了一些事件回调或 Composition API，比如 onEditorStateChange 之类。如果有，直接使用来订阅变化。
	•	如果没有，可以采取Vue内置方法，比如 watchPostEffect。在ProseKit组件挂载后，定期检查编辑器的某些状态值（例如插件状态）。
	•	另一种有效方法：在 ProseMirror 插件中，在其 apply 方法或 update 钩子里，利用我们注入的 Editor实例触发一个自定义事件（例如通过EventEmitter或简单地把需要的值赋给Vue reactive对象）。例如 Slash 插件每次候选列表变化时，调用 slashState.onUpdate(newState)，而我们在Vue中把slashState包装成ref，则UI会响应更新。
具体实现中，可以将需要响应的数据（如 Slash 当前是否显示、列表项数组）存在一个 reactive 的 ref 或 reactive 对象里，然后在插件逻辑改变时更新它。Vue组件直接使用这个 reactive 数据来源即可自动重渲。当然，也可使用 Pinia/Vuex 这类全局状态库。如果编辑器需与全局应用状态同步（例如变量值在其他组件修改，需要反映到编辑器中的某个节点属性），使用 Pinia 会方便很多，因其支持跨组件的状态共享和持久化。在我们的编辑器骨架中，大部分编辑器内部状态其实无需全局store，依赖 ProseMirror 本身的文档状态即可。但是上下文数据（如供 Mention 的外部数据源，或执行AI时传入的全局变量）可以放在 Pinia里，让编辑器插件去读取，从而把编辑器当作应用的一部分而非孤岛。
	•	组件事件联动：Vue3 组件之间通信可通过事件：比如 MentionSuggestions 组件在用户选中一个引用项时，可以通过 $emit('select', item) 通知父组件或直接调用编辑器命令插入节点。在我们的架构中，一般 SlashMenu 和 MentionSuggestions 这类组件会作为 Editor 容器组件的子组件浮层，可能并没有直接的父子关系到 Editor 容器。但由于都 inject 了 editor，我们可以不经由 Vue自带事件，而是直接调用 editor 的方法。举例来说，在 SlashMenuItem 子组件的点击事件处理里：

const editor = inject('editor');
const onClick = () => {
  editor?.commands?.insertNode(slashItem.nodeType, slashItem.options);
};

这样直接触发了编辑器命令，然后Slash插件自身会检测到选择完成从而关闭菜单。不经过额外的中间层。另外，对于 NodeView 组件和外围的交互，比如 IfBlockView 内部想打开属性侧栏编辑，可以通过 $emit冒泡到 ProseKit 容器，让容器去触发侧栏组件的显示，同时将当前节点引用传递过去。在容器组件中可以维护一个ref例如 activeNodeConfig 来决定属性编辑器内容。

	•	状态管理建议：综合而言，如果主要交互都在编辑器内部，我们尽量利用 ProseMirror/ProseKit 提供的 plugin state + Vue reactive 来管理状态，尽量避免引入全局状态复杂度。但如果涉及编辑器与应用其他部分的通信（例如运行Flow、触发后端请求），Pinia 可以帮助管理这些数据流。可以建立一个 useEditorStore，其中包括当前文档 JSON、光标位置、选中节点信息等调试或需要外部用到的状态，以及共享的上下文信息（比如当前用户输入集）。编辑器插件每当内容变更就更新 store 中的 doc（这个可通过监听 transaction 实现），那么外部组件就能通过 store 获取最新文档（例如用于预览JSON或保存）。反过来，外部改变 store 的某些值（如全局变量）也可通过 effect 影响编辑器（例如触发某插件去插入这些变量值）。

总之，在 Vue3 中集成 ProseKit，要充分运用组合式API和依赖注入来组织组件互动。ProseKit 提供的 Vue 封装简化了很多细节，如 NodeView 集成 ￼等，让我们更多关注业务本身。在这个架构下，添加新功能时，我们遵循：
	•	写扩展：定义好ProseMirror扩展（节点/插件/命令）。
	•	写组件：如有UI交互，就写对应Vue组件并通过 defineVueNodeView 或 context 插入机制注册。
	•	注册扩展：在编辑器初始化或组件setup时挂上扩展。
	•	调试交互：利用Vue devtools和Pinia devtools观察状态，确保事件联动正确。

这样，我们就可以逐步搭建出类似 Wordware.ai 的AI编辑器框架：基于 Vue3 + ProseKit，插件驱动，各模块松耦合可插拔。无论是基础的富文本功能，还是高级的逻辑编排，都通过扩展机制实现，符合现代前端架构的最佳实践，也为将来扩展智能提示、模型交互等功能留下了充足空间。

参考资料：ProseKit 官方文档和示例代码，Wordware 文档 ￼ ￼以及 ProseMirror 社区对 mentions/autocomplete 的讨论提供了实现这些功能的思路 ￼ ￼。通过整合这些经验，我们有信心实现一个功能媲美 Wordware 的编辑器内核，并且拥有良好的可扩展性和维护性。

---

参考以上内容，最小化实现功能。
