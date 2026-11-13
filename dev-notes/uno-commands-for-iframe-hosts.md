# UNO commands for iframe hosts

How a host application that embeds WYSIWYG-Editor in an iframe can drive the
document with `Send_UNO_Command`, plus a ranked list of the most useful
commands and a full reference table.

## How it works

```js
iframe.contentWindow.postMessage(JSON.stringify({
  MessageId: 'Send_UNO_Command',
  Values: {
    Command: '.uno:TrackChanges',
    Args: { TrackChanges: { type: 'boolean', value: true } }   // optional
  }
}), editorOrigin);
```

- Handler: `browser/src/map/handler/Map.WOPI.js` (`Send_UNO_Command`). It does **not**
  whitelist commands. Any command the engine knows is forwarded to the document.
- Before you send anything, send `Host_PostmessageReady` and wait for
  `App_LoadingStatus` with `Status: "Document_Loaded"`. Commands sent earlier are lost.
  `PostMessageOrigin` in CheckFileInfo must match your host origin.
- In read-only mode only a short whitelist works (`browser/src/control/Toolbar.js`,
  `sendUnoCommand`): Save, ExportToPDF, WordCountDialog, Navigator, SelectAll and, if
  commenting is allowed, the comment commands.
- **Toggles:** many commands flip a state, for example `TrackChanges`, `Bold` and
  `ShowTrackedChanges`. If you send them without arguments, the result depends on the
  current state. Wherever a boolean argument exists, set the state explicitly. The short
  form `'.uno:TrackChanges?TrackChanges:bool=true'` also works.
- To read state back, listen for `Doc_ModifiedStatus` or request state through
  `Get_Export_Formats` and similar messages. UNO state changes are not sent to the host
  by default.

## Ranked: most useful for an embedding host

### Tier 1 – review and workflow control (Writer, partly Calc)

| Command | What it does | Args / notes |
|---|---|---|
| `.uno:TrackChanges` | Turns change recording on or off. | `TrackChanges: boolean`. Always pass it: it is a toggle. |
| `.uno:ProtectTraceChangeMode` | Locks recording behind a password so users can't switch it off. | Opens a password dialog. Good for enforced review. |
| `.uno:ShowTrackedChanges` | Shows or hides tracked changes (markup vs. final view). | `ShowTrackedChanges: boolean` |
| `.uno:AcceptAllTrackedChanges` | Accepts every tracked change. | – |
| `.uno:RejectAllTrackedChanges` | Rejects every tracked change. | – |
| `.uno:AcceptTrackedChange` / `.uno:RejectTrackedChange` | Accepts or rejects the change at the cursor or selection. | – |
| `.uno:AcceptTrackedChangeToNext` / `.uno:RejectTrackedChangeToNext` | Accepts or rejects the change, then jumps to the next one. | – |
| `.uno:NextTrackedChange` / `.uno:PreviousTrackedChange` | Navigates between changes. | – |
| `.uno:AcceptTrackedChanges` | Opens the Manage Changes dialog (Writer). Calc uses `.uno:AcceptChanges`. | – |
| `.uno:TrackChangesInThisView` / `.uno:TrackChangesInAllViews` | Records changes for this user only or for all users (collaborative editing). | – |
| `.uno:Save` | Saves the document through WOPI. | `DontTerminateEdit: boolean`, `DontSaveIfUnmodified: boolean` |
| `.uno:SetDocumentProperties` | Opens the document properties dialog (title, author and so on). | – |

### Tier 2 – comments and export

| Command | What it does | Args / notes |
|---|---|---|
| `.uno:InsertAnnotation` | Inserts a comment. Without args the comment editor opens interactively. | `Text: string`; in Calc it is `.uno:InsertAnnotation` too |
| `.uno:ShowResolvedAnnotations` | Shows or hides resolved comments. | – |
| `.uno:DeleteAllNotes` | Deletes all comments. | – |
| `.uno:ExportToPDF` / `.uno:ExportDirectToPDF` | Exports to PDF (with the dialog / directly). | – |
| `.uno:ExportToEPUB` | Exports to EPUB. | – |
| `.uno:Print` | Prints. | – |
| `.uno:Signature` | Opens the digital signature dialog. | – |
| `.uno:WordCountDialog` | Shows word and character counts. | – |

### Tier 3 – content and positioning from the host

| Command | What it does | Args / notes |
|---|---|---|
| `.uno:InsertText` | Inserts text at the cursor. Useful for text blocks or placeholders coming from the host. | `Text: string` |
| `.uno:InsertBookmark` | Inserts a bookmark. | `Bookmark: string`, `BookmarkText: string` |
| `.uno:InsertField` | Inserts a field (date, page number, ...). | – |
| `.uno:GoToStartOfDoc` / `.uno:GoToEndOfDoc` | Moves the cursor to the start or end of the document. | – |
| `.uno:GotoPage` | Jumps to a page. | – |
| `.uno:SearchDialog` | Opens Find & Replace. | – |
| `.uno:ExecuteSearch` | Runs a search or replace without the dialog. | `SearchItem.SearchString`, `SearchItem.ReplaceString`, `SearchItem.Command` (0 = find, 3 = replace all) |
| `.uno:SelectAll` | Selects everything. | – |
| `.uno:Undo` / `.uno:Redo` | Undoes or redoes the last step. | – |
| `.uno:InsertPagebreak` | Inserts a page break. | – |
| `.uno:InsertTable` | Inserts a table. | `Columns: short`, `Rows: short` |
| `.uno:InsertGraphic` | Inserts an image (file dialog). | – |
| `.uno:InsertSymbol` | Inserts a special character. | – |
| `.uno:InsertMultiIndex`, `.uno:UpdateAllIndexes` | Inserts or updates a table of contents. | – |
| `.uno:UpdateFields` | Updates fields. | – |

### Tier 4 – UI and view control

| Command | What it does |
|---|---|
| `.uno:ToolbarMode?Mode:string=notebookbar_online.ui` / `...=Default` | Switches between the tabbed toolbar and the compact toolbar. |
| `.uno:SidebarShow` / `.uno:SidebarHide` | Shows or hides the sidebar. |
| `.uno:Navigator` | Opens or closes the Navigator. |
| `.uno:ShowRuler` | Shows or hides the ruler. |
| `.uno:ControlCodes` | Shows formatting marks (¶). |
| `.uno:SpellOnline` | Toggles automatic spell checking. |
| `.uno:LanguageStatus?Language:string=Default_German (Germany)` | Sets the document language. |
| `.uno:ZoomPlus` / `.uno:ZoomMinus` | **Not useful through postMessage.** Zoom happens in the browser client. These commands only zoom when they come from the menu/toolbar (client action `zoomin`/`zoomout`). Sent through `Send_UNO_Command`, they go to the engine and don't change the view. For an opening zoom, use `user_interface.smart_zoom` / `user_interface.default_zoom` in `coolwsd.xml`. |
| `.uno:FullScreen` | Switches to full screen. |
| `.uno:Presentation` | Starts the slideshow (Impress). |

### Tier 5 – formatting (usually left to the user)

`Bold`, `Italic`, `Underline`, `Strikeout`, `CharFontName`, `FontHeight`,
`Color`, `CharBackColor`, `LeftPara`/`CenterPara`/`RightPara`/`JustifyPara`,
`DefaultBullet`, `DefaultNumbering`, `StyleApply` (`Style: string`, `FamilyName: string`),
`SetDefault` (clears direct formatting), `FormatPaintbrush`, `PageDialog`, `FontDialog`,
`ParagraphDialog`. Calc-specific: `InsertRows*`, `DeleteRows`, `SortAscending`,
`DataFilterAutoFilter`, `FreezePanes`, `Protect` (protect sheet), `ToolProtectionDocument`.

## Full reference: every command the WYSIWYG-Editor UI uses

These are the 798 commands referenced in `browser/src`. The label comes from the engine
command configuration (`engine/officecfg/.../UI/*Commands.xcu`, en-US). "–" means the
engine has no label (usually internal or LOK-only commands). The engine supports even
more commands, the full LibreOffice dispatch list, but those are untested in the
online UI.

| Command | Label | App |
|---|---|---|
| `.uno:About` | – | – |
| `.uno:AcceptAllTrackedChanges` | Accept All | Calc, Writer |
| `.uno:AcceptChanges` | Manage... | Calc |
| `.uno:AcceptFormula` | – | – |
| `.uno:AcceptTrackedChange` | Accept | Calc, Writer |
| `.uno:AcceptTrackedChangeToNext` | Accept and Move to Next | Calc, Writer |
| `.uno:AcceptTrackedChanges` | Manage... | Writer |
| `.uno:AccessibilityCheck` | Accessibility Check... | Writer |
| `.uno:AddName` | Define Name... | Calc |
| `.uno:AddPrintArea` | Add | Calc |
| `.uno:AddSlideSection` | Add Section | Impress/Draw |
| `.uno:AddTheme` | Add Theme... | Alle |
| `.uno:AlignBlock` | Justified | Calc |
| `.uno:AlignBottom` | Align Bottom to Anchor | Calc, Writer |
| `.uno:AlignCenter` | Centered | Alle |
| `.uno:AlignDown` | Bottom | Alle |
| `.uno:AlignHorizontalCenter` | Center Horizontal | Calc, Writer |
| `.uno:AlignLeft` | Align Left | Calc, Writer |
| `.uno:AlignMiddle` | Middle | Alle |
| `.uno:AlignRight` | Align Right | Calc, Writer |
| `.uno:AlignTop` | Align Top to Anchor | Calc, Writer |
| `.uno:AlignUp` | Top | Alle |
| `.uno:AlignVCenter` | Center Vertically | Calc |
| `.uno:AnalysisOfVarianceDialog` | Analysis of Variance (ANOVA)... | Calc |
| `.uno:AnchorMenu` | Anchor Menu | Alle |
| `.uno:AnimationEffects` | Interaction... | Impress/Draw |
| `.uno:ArrangeMenu` | Arrange | Alle, Impress/Draw |
| `.uno:ArrowShapes` | Arrow Shapes | Alle |
| `.uno:AssignLayout` | Slide Layout | Impress/Draw |
| `.uno:AttributePageSize` | Page Size | Writer |
| `.uno:AutoCorrectDlg` | AutoCorrect Options... | Alle |
| `.uno:AutoFill` | AutoFill Data Series: automatic | Calc |
| `.uno:AutoFormat` | Table Styles | Alle |
| `.uno:AutoFormatApply` | Apply | Writer |
| `.uno:AutoFormatMenu` | AutoCorrect | Writer |
| `.uno:AutoFormatRedlineApply` | Apply and Edit Changes | Writer |
| `.uno:AutoSum` | Sum | Alle |
| `.uno:AutoSumMenu` | – | – |
| `.uno:BackgroundColor` | Background Color | Alle |
| `.uno:BasicShapes` | Basic Shapes | Alle |
| `.uno:Bezier_Unfilled` | Curve | Alle |
| `.uno:Bold` | Bold | Alle |
| `.uno:Bookmark` | – | – |
| `.uno:Bookmarks` | – | – |
| `.uno:BorderDialog` | Borders | Writer |
| `.uno:BorderInner` | – | – |
| `.uno:BorderOuter` | – | – |
| `.uno:BottomPaneImpress` | Notes Pane | Impress/Draw |
| `.uno:BringToFront` | Bring to Front | Alle |
| `.uno:CalcPageMargin` | – | – |
| `.uno:CalcPageSize` | – | – |
| `.uno:Calculate` | Recalculate | Calc |
| `.uno:CalculateHard` | Recalculate Hard | Calc |
| `.uno:CalculateSheet` | Recalculate Sheet | Calc |
| `.uno:CalculatedFieldRun` | Calculated Field | Calc |
| `.uno:CalloutShapes` | Callout Shapes | Alle |
| `.uno:Cancel` | Cancel | Calc |
| `.uno:CanvasPageCenter` | – | – |
| `.uno:CanvasPageVisArea` | – | – |
| `.uno:CanvasPreviewRects` | – | – |
| `.uno:CapturePoint` | Snap Guide... | Impress/Draw |
| `.uno:CellCursor` | – | – |
| `.uno:CellTextDlg` | – | – |
| `.uno:CellVertBottom` | Bottom | Alle |
| `.uno:CellVertCenter` | Center | Alle |
| `.uno:CellVertTop` | Top | Alle |
| `.uno:CenterPara` | Center | Alle |
| `.uno:ChangeAlignment` | Alignment... | Math |
| `.uno:ChangeBezier` | To Curve | Impress/Draw |
| `.uno:ChangeCaseRotateCase` | Cycle Case | Alle |
| `.uno:ChangeCaseToLower` | lowercase | Alle |
| `.uno:ChangeCaseToSentenceCase` | Sentence case | Alle |
| `.uno:ChangeCaseToTitleCase` | Capitalize Every Word | Alle |
| `.uno:ChangeCaseToToggleCase` | tOGGLE cASE | Alle |
| `.uno:ChangeCaseToUpper` | UPPERCASE | Alle |
| `.uno:ChangeDistance` | Spacing... | Math |
| `.uno:ChangeFont` | Fonts... | Math |
| `.uno:ChangeFontSize` | Font Size... | Math |
| `.uno:ChangeTabStop` | – | – |
| `.uno:ChangeTheme` | Dark Mode | Alle |
| `.uno:ChangesMenu` | Track Changes | Alle |
| `.uno:ChapterNumberingDialog` | Heading Numbering... | Writer |
| `.uno:CharBackColor` | Character Highlighting Color | Alle |
| `.uno:CharBackgroundExt` | – | – |
| `.uno:CharFontName` | Font Name | Alle |
| `.uno:CharSpacing` | – | – |
| `.uno:CharmapControl` | Symbol | Alle |
| `.uno:ChartSaveToNewTheme` | – | – |
| `.uno:ChiSquareTestDialog` | Chi-square Test... | Calc |
| `.uno:ClearOutline` | Remove Outline | Alle |
| `.uno:ClearTableStyle` | Clear Style | Calc |
| `.uno:CloseMasterView` | Close Master View | Impress/Draw |
| `.uno:CloseWin` | Close Window | Alle |
| `.uno:Color` | Font Color | Alle |
| `.uno:ColorScaleFormatDialog` | Conditional Formatting: Color Scale | Calc |
| `.uno:ColumnOperations` | Column | Calc |
| `.uno:ColumnWidth` | Column Width... | Calc |
| `.uno:Command` | – | – |
| `.uno:CommentChange` | Comment... | Calc |
| `.uno:CommentChangeTracking` | Comment... | Writer |
| `.uno:CommonAlignBottom` | Bottom | Alle |
| `.uno:CommonAlignHorizontalCenter` | Centered | Alle |
| `.uno:CommonAlignJustified` | Justified | Alle |
| `.uno:CommonAlignLeft` | Left | Alle |
| `.uno:CommonAlignRight` | Right | Alle |
| `.uno:CommonAlignTop` | Top | Alle |
| `.uno:CommonAlignVerticalCenter` | Center | Alle |
| `.uno:CompareDocuments` | Compare | Alle |
| `.uno:CompressGraphic` | Compress Image | Alle, Impress/Draw |
| `.uno:CondDateFormatDialog` | Conditional Formatting: Date | Calc |
| `.uno:ConditionalFormatDialog` | Conditional Formatting: More Rules | Calc |
| `.uno:ConditionalFormatEasy` | – | – |
| `.uno:ConditionalFormatManagerDialog` | Manage... | Calc |
| `.uno:ConditionalFormatMenu` | Conditional | Calc |
| `.uno:Connector` | Connector | Alle |
| `.uno:ConnectorArrowEnd` | Connector Ends with Arrow | Alle |
| `.uno:ConnectorArrows` | Connector with Arrows | Alle |
| `.uno:ConnectorCurve` | Curved Connector | Alle |
| `.uno:ConnectorCurveArrowEnd` | Curved Connector Ends with Arrow | Alle |
| `.uno:ConnectorCurveArrows` | Curved Connector with Arrows | Alle |
| `.uno:ConnectorLine` | Straight Connector | Alle |
| `.uno:ConnectorLineArrowEnd` | Straight Connector ends with Arrow | Alle |
| `.uno:ConnectorLineArrows` | Straight Connector with Arrows | Alle |
| `.uno:ConnectorLines` | Line Connector | Alle |
| `.uno:ConnectorLinesArrowEnd` | Line Connector Ends with Arrow | Alle |
| `.uno:ConnectorToolbox` | Connectors | Alle |
| `.uno:ContentControlProperties` | Content Control Properties | Writer |
| `.uno:Context` | Current Context | Alle |
| `.uno:ContinueNumbering` | Add to List | Writer |
| `.uno:ContourDialog` | Edit Contour... | Alle |
| `.uno:ControlCodes` | Formatting Marks | Writer |
| `.uno:ConvertCalcTableToRange` | Convert to Range | Calc |
| `.uno:ConvertFormulaToValue` | Formula to Value | Calc |
| `.uno:Copy` | Copy | Alle |
| `.uno:CopyHyperlinkLocation` | Copy Hyperlink Location | Alle |
| `.uno:CopyObjects` | Duplicate... | Impress/Draw |
| `.uno:CopySlide` | – | – |
| `.uno:CopyTab` | – | – |
| `.uno:CorrelationDialog` | Correlation... | Calc |
| `.uno:CovarianceDialog` | Covariance... | Calc |
| `.uno:Crop` | Crop | Alle |
| `.uno:CurrentTrackedChangeId` | – | – |
| `.uno:CursorBlinkTime` | – | – |
| `.uno:CustomAnimation` | Animation | Impress/Draw |
| `.uno:Cut` | Cut | Alle |
| `.uno:DataAreaRefresh` | Refresh Range | Calc |
| `.uno:DataBarFormatDialog` | Conditional Formatting: Data Bar | Calc |
| `.uno:DataDataPilotRun` | Pivot Table | Calc |
| `.uno:DataFilterAutoFilter` | AutoFilter | Calc |
| `.uno:DataFilterHideAutoFilter` | Hide AutoFilter | Calc |
| `.uno:DataFilterRemoveFilter` | Reset Filter | Calc |
| `.uno:DataFilterSpecialFilter` | Advanced Filter... | Calc |
| `.uno:DataFilterStandardFilter` | Standard Filter... | Calc |
| `.uno:DataMenu` | Data | Alle |
| `.uno:DataPilotMenu` | Pivot Table | Calc |
| `.uno:DataRanges` | Data Ranges... | Chart |
| `.uno:DataSelect` | Selection List | Calc |
| `.uno:DataSort` | Sort... | Calc |
| `.uno:DataTableMenu` | Table | Calc |
| `.uno:DatabaseSettings` | Table Style Options | Calc |
| `.uno:DecrementIndent` | Decrease | Alle |
| `.uno:DecrementLevel` | Demote Outline Level | Writer |
| `.uno:DecrementSubLevels` | Demote Outline Level with Subpoints | Writer |
| `.uno:DefaultBullet` | Unordered List | Alle |
| `.uno:DefaultNumbering` | Ordered List | Alle |
| `.uno:DefineDBName` | Define Range... | Calc |
| `.uno:DefineName` | Manage Names... | Calc |
| `.uno:DefinePrintArea` | Print Area | Calc |
| `.uno:Delete` | Delete Contents... | Alle, Calc |
| `.uno:DeleteAllAnnotation` | Delete All Comments | Impress/Draw |
| `.uno:DeleteAllNotes` | Delete All Comments | Alle, Calc |
| `.uno:DeleteAnnotation` | Delete Comment | Impress/Draw |
| `.uno:DeleteBookmark` | – | – |
| `.uno:DeleteBookmarks` | – | – |
| `.uno:DeleteCanvasSlide` | Delete Canvas Slide | Impress/Draw |
| `.uno:DeleteCell` | Delete Cells... | Calc |
| `.uno:DeleteColumns` | Delete Columns | Alle, Impress/Draw, Writer |
| `.uno:DeleteComment` | Delete Comment | Alle |
| `.uno:DeleteCommentThread` | Delete Comment Thread | Alle |
| `.uno:DeleteField` | – | – |
| `.uno:DeleteFields` | – | – |
| `.uno:DeleteNote` | Delete Comment | Calc |
| `.uno:DeletePage` | Delete Page | Impress/Draw |
| `.uno:DeletePivotTable` | Delete Pivot Table | Calc |
| `.uno:DeletePreviewShape` | – | – |
| `.uno:DeletePrintArea` | Clear | Calc |
| `.uno:DeleteRows` | Delete Rows | Calc, Impress/Draw, Writer |
| `.uno:DeleteSection` | – | – |
| `.uno:DeleteSections` | – | – |
| `.uno:DeleteSlide` | Delete Slide | Impress/Draw |
| `.uno:DeleteSparkline` | Delete Sparkline | Calc |
| `.uno:DeleteSparklineGroup` | Delete Sparkline Group | Calc |
| `.uno:DeleteTable` | Delete Table | Impress/Draw, Writer |
| `.uno:DeleteTextFormField` | – | – |
| `.uno:DeleteTextFormFields` | – | – |
| `.uno:DescriptiveStatisticsDialog` | Descriptive Statistics... | Calc |
| `.uno:DiagramArea` | Chart Area... | Chart |
| `.uno:DiagramAxisX` | X Axis... | Chart |
| `.uno:DiagramAxisY` | Y Axis... | Chart |
| `.uno:DiagramData` | Data Table... | Chart |
| `.uno:DiagramType` | Chart Type... | Chart |
| `.uno:DiagramWall` | Chart Wall... | Chart |
| `.uno:DistributeColumns` | Distribute Columns Evenly | Impress/Draw, Writer |
| `.uno:DistributeHorzCenter` | Distribute Horizontally Center | Alle |
| `.uno:DistributeHorzDistance` | Distribute Horizontally Spacing | Alle |
| `.uno:DistributeHorzLeft` | Distribute Horizontally Left | Alle |
| `.uno:DistributeHorzRight` | Distribute Horizontally Right | Alle |
| `.uno:DistributeRows` | Distribute Rows Evenly | Impress/Draw, Writer |
| `.uno:DistributeSelection` | Distribution | Alle |
| `.uno:DistributeVertBottom` | Distribute Vertically Bottom | Alle |
| `.uno:DistributeVertCenter` | Distribute Vertically Center | Alle |
| `.uno:DistributeVertDistance` | Distribute Vertically Spacing | Alle |
| `.uno:DistributeVertTop` | Distribute Vertically Top | Alle |
| `.uno:DocumentRepair` | – | – |
| `.uno:DownSearch` | Find Next  (F3) | Alle |
| `.uno:DownloadSignature` | – | – |
| `.uno:DrawFromTiles` | – | – |
| `.uno:DrawText` | Text Box | Alle |
| `.uno:DuplicatePage` | Duplicate Page | Impress/Draw |
| `.uno:DuplicateSheet` | Duplicate Sheet | Calc |
| `.uno:DuplicateSlide` | Duplicate Slide | Impress/Draw |
| `.uno:DuplicateTableStyle` | Duplicate Style | Calc |
| `.uno:EditAnnotation` | Edit Comment | Calc |
| `.uno:EditDiagram` | Edit Diagram | Alle |
| `.uno:EditDoc` | Edit Mode | Alle |
| `.uno:EditHeaderAndFooter` | Headers and Footers... | Calc |
| `.uno:EditHyperlink` | Hyperlink | Alle |
| `.uno:EditMenu` | Edit | Alle |
| `.uno:EditPrintArea` | Edit | Calc |
| `.uno:EditRegion` | Sections... | Writer |
| `.uno:EditSparkline` | Edit Sparkline... | Calc |
| `.uno:EditSparklineGroup` | Edit Sparkline Group... | Calc |
| `.uno:EmojiControl` | – | – |
| `.uno:EnterGroup` | Enter Group | Alle |
| `.uno:EntireCell` | Select Cell | Writer |
| `.uno:EntireColumn` | Select Column | Impress/Draw, Writer |
| `.uno:EntireRow` | Select Row | Impress/Draw, Writer |
| `.uno:ExecuteSearch` | – | – |
| `.uno:ExitSheetView` | Exit Sheet View | Calc |
| `.uno:ExpandGlossary` | Run AutoText Entry | Writer |
| `.uno:ExponentialSmoothingDialog` | Exponential Smoothing... | Calc |
| `.uno:ExportDirectToPDF` | Export Directly as PDF | Alle |
| `.uno:ExportToEPUB` | EPUB | Alle |
| `.uno:ExportToPDF` | Export as PDF | Alle |
| `.uno:Extrusion3DColor` | 3D Color | Alle |
| `.uno:ExtrusionDepthFloater` | Depth | Alle |
| `.uno:ExtrusionDirectionFloater` | Direction | Alle |
| `.uno:ExtrusionLightingFloater` | Lighting | Alle |
| `.uno:ExtrusionSurfaceFloater` | Surface | Alle |
| `.uno:FTestDialog` | F-test... | Calc |
| `.uno:Feedback` | – | – |
| `.uno:Field` | – | – |
| `.uno:Fields` | Fields | Writer |
| `.uno:FillColor` | Fill Color | Alle |
| `.uno:FillGradient` | – | – |
| `.uno:FillPageColor` | – | – |
| `.uno:FillPageGradient` | – | – |
| `.uno:FillShadow` | Shadow | Alle |
| `.uno:FillShadowColor` | – | – |
| `.uno:FilterMenu` | More Filters | Calc |
| `.uno:FlipHorizontal` | Flip Horizontally | Alle |
| `.uno:FlipVertical` | Flip Vertically | Alle |
| `.uno:FlowChartShapes` | Flowchart Shapes | Alle |
| `.uno:FontColor` | Font Color | Writer |
| `.uno:FontDialog` | Character... | Alle |
| `.uno:FontHeight` | Font Size | Alle |
| `.uno:FontworkAlignmentFloater` | Fontwork Alignment | Alle |
| `.uno:FontworkCharacterSpacingFloater` | Fontwork Character Spacing | Alle |
| `.uno:FontworkGalleryFloater` | Insert Fontwork | Alle |
| `.uno:FontworkShapeType` | Fontwork Shape | Alle |
| `.uno:FootnoteDialog` | Footnote/Endnote Settings... | Writer |
| `.uno:FormatArea` | Area... | Alle |
| `.uno:FormatBulletsMenu` | Lists | Alle |
| `.uno:FormatCellBorders` | Format Cell Borders | Calc |
| `.uno:FormatCellDialog` | Format Cells... | Calc |
| `.uno:FormatColumns` | Columns... | Writer |
| `.uno:FormatFormMenu` | Form | Alle |
| `.uno:FormatGroup` | Group | Alle |
| `.uno:FormatLine` | Line... | Alle |
| `.uno:FormatMenu` | Format | Alle |
| `.uno:FormatPaintbrush` | Clone | Alle, Writer |
| `.uno:FormatSelection` | Format Selection... | Chart |
| `.uno:FormatSpacingMenu` | Spacing | Alle |
| `.uno:FormatSparklineMenu` | Sparklines | Calc |
| `.uno:FormatTextMenu` | Text | Alle |
| `.uno:FormatUngroup` | Ungroup | Alle |
| `.uno:FormattingMarkMenu` | Formatting Mark | Alle |
| `.uno:FormulaDepChain` | – | – |
| `.uno:ForumHelp` | – | – |
| `.uno:FourierAnalysisDialog` | Fourier Analysis... | Calc |
| `.uno:FrameLineColor` | Border Color | Alle |
| `.uno:FreezeCellsMenu` | Freeze Cells | Calc |
| `.uno:FreezePanes` | Freeze Panes | Calc |
| `.uno:FreezePanesColumn` | Freeze First Column | Calc |
| `.uno:FreezePanesRow` | Freeze First Row | Calc |
| `.uno:FullScreen` | Full Screen | Alle |
| `.uno:FunctionBox` | Function List | Calc |
| `.uno:FunctionDialog` | Function... | Calc |
| `.uno:Gallery` | Gallery | Alle |
| `.uno:GetDesignTemplates` | – | – |
| `.uno:GlowEffect` | – | – |
| `.uno:GlueEditMode` | Gluepoints | Impress/Draw |
| `.uno:GoToCell` | – | – |
| `.uno:GoalSeekDialog` | Goal Seek... | Calc |
| `.uno:GotoOutlineIndex` | – | – |
| `.uno:GotoPage` | Go to Page... | Impress/Draw, Writer |
| `.uno:GotoSlide` | Go to Slide... | Impress/Draw |
| `.uno:GrafContrast` | Contrast | Alle |
| `.uno:GrafLuminance` | Brightness | Alle |
| `.uno:GrafMode` | Image Mode | Alle |
| `.uno:GrafTransparence` | Transparency | Alle |
| `.uno:GraphicDialog` | Image Properties | Writer |
| `.uno:GraphicFilterInvert` | Invert | Alle |
| `.uno:GraphicFilterPopart` | Pop Art | Alle |
| `.uno:GraphicFilterRemoveNoise` | Remove Noise | Alle |
| `.uno:GraphicFilterSharpen` | Sharpen | Alle |
| `.uno:GraphicFilterSobel` | Charcoal Sketch | Alle |
| `.uno:GraphicFilterToolbox` | Filter | Alle |
| `.uno:GridUse` | Snap to Grid | Alle |
| `.uno:GridVisible` | Grid | Alle |
| `.uno:Group` | Group... | Alle |
| `.uno:GroupOutlineMenu` | Group and Outline | Calc |
| `.uno:GroupSparklines` | Group Sparklines | Calc |
| `.uno:Grow` | Increase | Alle |
| `.uno:HandleDuplicateRecords` | Duplicates... | Calc |
| `.uno:HasOverviewPage` | – | – |
| `.uno:HeaderAndFooter` | Header and Footer... | Impress/Draw |
| `.uno:Headings` | – | – |
| `.uno:HelpMenu` | Help | Alle |
| `.uno:Hide` | Hide Sheet | Calc |
| `.uno:HideAllNotes` | Hide All Comments | Calc |
| `.uno:HideColumn` | Hide Columns | Calc |
| `.uno:HideDetail` | Hide Details | Alle |
| `.uno:HideRow` | Hide Rows | Calc |
| `.uno:HideSlide` | Hide Slide | Impress/Draw |
| `.uno:HyperlinkDialog` | Hyperlink... | Alle |
| `.uno:Hyphenate` | Hyphenation... | Calc, Writer |
| `.uno:Hyphenation` | Hyphenation | Impress/Draw |
| `.uno:IconSetFormatDialog` | Conditional Formatting: Icon Set | Calc |
| `.uno:IncrementIndent` | Increase | Alle |
| `.uno:IncrementLevel` | Promote Outline Level | Writer |
| `.uno:IncrementSubLevels` | Promote Outline Level with Subpoints | Writer |
| `.uno:IndexesMenu` | Table of Contents and Index | Writer |
| `.uno:Insert` | Insert Sheet... | Calc |
| `.uno:InsertAVMedia` | Media | Alle |
| `.uno:InsertAnnotation` | Comment | Alle, Calc |
| `.uno:InsertAuthorField` | First Author | Impress/Draw, Writer |
| `.uno:InsertBookmark` | Bookmark... | Writer |
| `.uno:InsertBreak` | Manual Break... | Writer |
| `.uno:InsertCalcTable` | Table... | Calc |
| `.uno:InsertCalculation` | Calculated Field... | Calc |
| `.uno:InsertCanvasSlide` | New Canvas Slide | Impress/Draw |
| `.uno:InsertCaptionDialog` | Caption... | Writer |
| `.uno:InsertCell` | Insert Cells... | Calc |
| `.uno:InsertCheckboxContentControl` | Check Box Content Control | Writer |
| `.uno:InsertColumn` | – | – |
| `.uno:InsertColumnBreak` | Insert Column Break | Calc, Writer |
| `.uno:InsertColumns` | Insert Column | Alle |
| `.uno:InsertColumnsAfter` | Insert Columns After | Calc, Impress/Draw, Writer |
| `.uno:InsertColumnsBefore` | Insert Columns Before | Calc, Impress/Draw, Writer |
| `.uno:InsertColumnsMenu` | Insert Columns | Calc |
| `.uno:InsertContentControl` | Rich Text Content Control | Writer |
| `.uno:InsertCurrentDate` | Insert Current Date | Calc |
| `.uno:InsertCurrentTime` | Insert Current Time | Calc |
| `.uno:InsertDateContentControl` | Date Content Control | Writer |
| `.uno:InsertDateField` | Date (fixed) | Writer |
| `.uno:InsertDateFieldFix` | Date (fixed) | Impress/Draw |
| `.uno:InsertDateFieldVar` | Date (variable) | Impress/Draw, Writer |
| `.uno:InsertDraw` | Draw Functions | Alle |
| `.uno:InsertDropdownContentControl` | Drop-Down List Content Control | Writer |
| `.uno:InsertEndnote` | Endnote | Writer |
| `.uno:InsertField` | More Fields... | Writer |
| `.uno:InsertFieldCtrl` | Field | Writer |
| `.uno:InsertFootnote` | Footnote | Writer |
| `.uno:InsertFormula` | Edit Formula | Writer |
| `.uno:InsertFrame` | Frame | Writer |
| `.uno:InsertFunction` | Insert Function | Calc |
| `.uno:InsertGraphic` | Image... | Alle |
| `.uno:InsertHardHyphen` | Insert Non-breaking Hyphen | Alle |
| `.uno:InsertHeaderFooterMenu` | Header and Footer | Writer |
| `.uno:InsertIndexesEntry` | Insert Index Entry | Writer |
| `.uno:InsertLRM` | Insert Left-to-Right Mark | Alle |
| `.uno:InsertMath` | Formula Object... | Alle |
| `.uno:InsertMenu` | Insert | Alle |
| `.uno:InsertMenuTitles` | Titles... | Chart |
| `.uno:InsertMode` | Insert Mode | Alle |
| `.uno:InsertMultiIndex` | Table of Contents | Writer |
| `.uno:InsertNeutralParagraph` | Insert Unnumbered Entry | Writer |
| `.uno:InsertNonBreakingSpace` | Insert No-break Space | Alle |
| `.uno:InsertObject` | Insert OLE Object | Alle |
| `.uno:InsertObjectChart` | Chart... | Alle |
| `.uno:InsertObjectStarMath` | Formula Object... | Calc, Writer |
| `.uno:InsertPage` | New Page | Impress/Draw |
| `.uno:InsertPageCountField` | Page Count | Writer |
| `.uno:InsertPageField` | Page Number | Impress/Draw |
| `.uno:InsertPageFooter` | Footer | Writer |
| `.uno:InsertPageHeader` | Header | Writer |
| `.uno:InsertPageNumberField` | Page Number Field | Writer |
| `.uno:InsertPageTitleField` | Page Title | Impress/Draw |
| `.uno:InsertPagebreak` | Page Break | Writer |
| `.uno:InsertPagesField` | Page Count | Impress/Draw |
| `.uno:InsertPictureContentControl` | Picture Content Control | Writer |
| `.uno:InsertPivotTable` | Insert or Edit... | Calc |
| `.uno:InsertQrCode` | QR and Barcode... | Alle |
| `.uno:InsertRLM` | Insert Right-to-Left Mark | Alle |
| `.uno:InsertReferenceField` | Cross-reference... | Writer |
| `.uno:InsertRows` | Insert Row | Alle |
| `.uno:InsertRowsAfter` | Insert Rows Below | Calc, Impress/Draw, Writer |
| `.uno:InsertRowsBefore` | Insert Rows Above | Calc, Impress/Draw, Writer |
| `.uno:InsertRowsMenu` | Insert Rows | Calc |
| `.uno:InsertSection` | Section... | Writer |
| `.uno:InsertShapeIntoSlide` | – | – |
| `.uno:InsertSignatureLine` | Signature Line... | Alle |
| `.uno:InsertSlide` | New Slide | Impress/Draw |
| `.uno:InsertSlideField` | Slide Number | Impress/Draw |
| `.uno:InsertSlideTitleField` | Slide Title | Impress/Draw |
| `.uno:InsertSlidesField` | Slide Count | Impress/Draw |
| `.uno:InsertSoftHyphen` | Insert Soft Hyphen | Alle |
| `.uno:InsertSparkline` | Sparkline... | Calc |
| `.uno:InsertSymbol` | Symbol | Alle |
| `.uno:InsertTable` | Table | Impress/Draw, Writer |
| `.uno:InsertThreadedComment` | Comment | Calc |
| `.uno:InsertTimeField` | Time Field | Alle, Writer |
| `.uno:InsertTimeFieldFix` | Time (fixed) | Impress/Draw |
| `.uno:InsertTimeFieldVar` | Time (variable) | Impress/Draw, Writer |
| `.uno:InsertTitleField` | Title | Writer |
| `.uno:InsertTopicField` | Subject | Writer |
| `.uno:InsertWJ` | Insert Word Joiner | Alle |
| `.uno:InsertZWSP` | Insert Zero-width Space | Alle |
| `.uno:InvertBackground` | – | – |
| `.uno:Italic` | Italic | Alle |
| `.uno:JumpDownThisLevel` | To Next Paragraph in Level | Writer |
| `.uno:JumpToMark` | – | – |
| `.uno:JumpToNextFrame` | To Next Frame | Writer |
| `.uno:JumpToNextTable` | To Next Table | Calc, Writer |
| `.uno:JumpToPrevTable` | To Previous Table | Calc, Writer |
| `.uno:JumpToTable` | Go to Sheet... | Calc |
| `.uno:JumpUpThisLevel` | To Previous Paragraph in Level | Writer |
| `.uno:JustifyPara` | Justified | Alle |
| `.uno:KeyboardShortcuts` | – | – |
| `.uno:KitSidebarWriterPage` | – | – |
| `.uno:LanguageMenu` | Language | Alle |
| `.uno:LanguageStatus` | Language Status | Alle |
| `.uno:LatestUpdates` | – | – |
| `.uno:LayerVisibility` | – | – |
| `.uno:LeaveGroup` | Exit Group | Alle |
| `.uno:LeavePreviewEdit` | – | – |
| `.uno:LeftPara` | Left | Alle |
| `.uno:LeftRightParaMargin` | – | – |
| `.uno:Legend` | Format Legend | Chart |
| `.uno:Line` | Line | Alle |
| `.uno:LineEndStyle` | Arrow Style | Alle |
| `.uno:LineNumberingDialog` | Line Numbering... | Writer |
| `.uno:LineSpacing` | Line Spacing | Alle |
| `.uno:LineStyle` | Border Style | Alle |
| `.uno:LineTransparence` | Line Transparency | Alle |
| `.uno:LineWidth` | Line Thickness | Alle |
| `.uno:MasterSlidesPanel` | Master Slides | Impress/Draw |
| `.uno:MergeCells` | Merge Cells | Alle |
| `.uno:MergeDocuments` | Merge | Alle |
| `.uno:ModifiedStatus` | Document Modified | Alle |
| `.uno:ModifyPage` | Slide Layout | Impress/Draw |
| `.uno:Move` | Move or Copy Sheet... | Calc |
| `.uno:MoveDown` | Move Item Down | Writer |
| `.uno:MoveDownSubItems` | Move Item Down with Subpoints | Writer |
| `.uno:MoveKeepInsertMode` | – | – |
| `.uno:MovePreviewShape` | – | – |
| `.uno:MovePreviewShapeOut` | – | – |
| `.uno:MoveShapeHandle` | – | – |
| `.uno:MoveSlideSectionDown` | Move Section Down | Impress/Draw |
| `.uno:MoveSlideSectionUp` | Move Section Up | Impress/Draw |
| `.uno:MoveUp` | Move Item Up | Writer |
| `.uno:MoveUpSubItems` | Move Item Up with Subpoints | Writer |
| `.uno:MovingAverageDialog` | Moving Average... | Calc |
| `.uno:Name` | Rename Sheet | Calc |
| `.uno:NameGroup` | Name... | Alle |
| `.uno:NamesMenu` | Named Ranges and Expressions | Calc |
| `.uno:Navigator` | Navigator | Alle |
| `.uno:NavigatorSelectComment` | Navigator select comment. | Alle |
| `.uno:NewSheetView` | New Sheet View | Calc |
| `.uno:NewTableStyle` | New Table Style... | Calc |
| `.uno:NextAnnotation` | Next Comment | Impress/Draw |
| `.uno:NextDocumentTab` | – | – |
| `.uno:NextTrackedChange` | Next | Calc, Writer |
| `.uno:NormalMultiPaneGUI` | Normal View | Impress/Draw |
| `.uno:NoteVisible` | Show Comment | Calc |
| `.uno:NotesMode` | Notes | Impress/Draw |
| `.uno:NumberFormatCurrency` | Number Format: Currency | Calc, Writer |
| `.uno:NumberFormatDate` | Number Format: Date | Calc, Writer |
| `.uno:NumberFormatDecDecimals` | Delete Decimal Place | Calc |
| `.uno:NumberFormatDecimal` | Number Format: Decimal | Calc, Writer |
| `.uno:NumberFormatIncDecimals` | Add Decimal Place | Calc |
| `.uno:NumberFormatMenu` | Number Format | Calc |
| `.uno:NumberFormatPercent` | Number Format: Percent | Calc, Writer |
| `.uno:NumberFormatScientific` | Number Format: Exponential | Calc, Writer |
| `.uno:NumberFormatStandard` | Number Format: Standard | Calc, Writer |
| `.uno:NumberFormatThousands` | Thousands Separator | Calc |
| `.uno:NumberFormatTime` | Number Format: Time | Calc, Writer |
| `.uno:NumberingStart` | Restart Numbering | Writer |
| `.uno:ObjectAlign` | Align Objects | Alle |
| `.uno:ObjectAlignLeft` | Left | Alle |
| `.uno:ObjectAlignRight` | Right | Alle |
| `.uno:ObjectBackOne` | Back One | Alle |
| `.uno:ObjectForwardOne` | Forward One | Alle |
| `.uno:ObjectTitleDescription` | Alt Text... | Alle, Chart |
| `.uno:Objects3DToolbox` | 3D Objects | Impress/Draw |
| `.uno:OnlineAutoFormat` | While Typing | Writer |
| `.uno:OnlineHelp` | – | – |
| `.uno:Open` | Open... | Alle |
| `.uno:OpenHyperlink` | – | – |
| `.uno:OpenHyperlinkOnCursor` | Open Hyperlink | Alle |
| `.uno:OptimizeTable` | Optimize Size | Impress/Draw, Writer |
| `.uno:Orientation` | Orientation | Writer |
| `.uno:OutlineBullet` | Bullets and Numbering... | Alle |
| `.uno:OutlineFont` | Outline Font Effect | Alle |
| `.uno:Overline` | Overline | Alle |
| `.uno:PageDialog` | Page Style... | Writer |
| `.uno:PageDown` | Next Page | Writer |
| `.uno:PageFormatDialog` | Format Page... | Calc |
| `.uno:PageLRMargin` | – | – |
| `.uno:PageLinks` | – | – |
| `.uno:PageMargin` | Page Margins | Writer |
| `.uno:PageMenu` | Page | Impress/Draw |
| `.uno:PageNumberWizard` | Page Number... | Writer |
| `.uno:PageSettingDialog` | Page Settings - Paper format | Writer |
| `.uno:PageSetup` | Format Page | Impress/Draw |
| `.uno:PageStatus` | Slide/Layer | Impress/Draw |
| `.uno:PageULMargin` | – | – |
| `.uno:PageUp` | Previous Page | Writer |
| `.uno:ParaLeftToRight` | Left-To-Right | Alle |
| `.uno:ParaRightToLeft` | Right-To-Left | Alle |
| `.uno:ParaRightToLft` | – | – |
| `.uno:ParagraphDialog` | Paragraph... | Alle |
| `.uno:ParaspaceDecrease` | Decrease | Alle |
| `.uno:ParaspaceIncrease` | Increase | Alle |
| `.uno:Paste` | Paste | Alle |
| `.uno:PasteSpecial` | Paste Special... | Calc, Impress/Draw, Writer |
| `.uno:PasteSpecialMenu` | Paste Special | Alle |
| `.uno:PickList` | File | Alle |
| `.uno:Position` | – | – |
| `.uno:PrepareSignature` | – | – |
| `.uno:PresentationCleanup` | – | – |
| `.uno:PresentationCurrentSlide` | Start from Current Slide | Impress/Draw |
| `.uno:PresentationLayout` | Change Slide Master... | Impress/Draw |
| `.uno:PreviewEditMode` | – | – |
| `.uno:PreviewShapeSelection` | – | – |
| `.uno:PreviousAnnotation` | Previous Comment | Impress/Draw |
| `.uno:PreviousDocumentTab` | – | – |
| `.uno:PreviousTrackedChange` | Previous | Calc, Writer |
| `.uno:PrincipalComponentAnalysisDialog` | Principal Component Analysis... | Calc |
| `.uno:Print` | Print... | Alle |
| `.uno:PrintRangesMenu` | Print Ranges | Calc |
| `.uno:PromoteComment` | Promote Comment To Root | Alle |
| `.uno:Protect` | Protect Cells | Calc, Writer |
| `.uno:ProtectTraceChangeMode` | Protect... | Alle |
| `.uno:QuickFind` | – | – |
| `.uno:Quit` | Exit | Alle |
| `.uno:RecalcPivotTable` | Refresh Pivot Table | Calc |
| `.uno:RecentlyUsedDocumentTab` | – | – |
| `.uno:RedactDoc` | Redact | Alle |
| `.uno:RedlineRenderMode` | – | – |
| `.uno:Redo` | Redo | Alle |
| `.uno:RegressionDialog` | Regression... | Calc |
| `.uno:ReinstateTrackedChange` | Reject but Track | Writer |
| `.uno:RejectAllTrackedChanges` | Reject All | Calc, Writer |
| `.uno:RejectTextEditRequest` | – | – |
| `.uno:RejectTrackedChange` | Reject | Calc, Writer |
| `.uno:RejectTrackedChangeToNext` | Reject and Move to Next | Calc, Writer |
| `.uno:RemoteGrammarCheckers` | – | – |
| `.uno:Remove` | Delete Sheet... | Calc |
| `.uno:RemoveBullets` | No List | Writer |
| `.uno:RemoveCalcTable` | Remove Table... | Calc |
| `.uno:RemoveDuplicates` | Remove Duplicates... | Calc |
| `.uno:RemoveHyperlink` | Remove Hyperlink | Alle |
| `.uno:RemoveSheetView` | Remove Sheet View | Calc |
| `.uno:RemoveSlideSection` | Remove Section | Impress/Draw |
| `.uno:RemoveSlideSectionAndSlides` | Remove Section & Slides | Impress/Draw |
| `.uno:RenameCalcTable` | Rename Table... | Calc |
| `.uno:RenameMasterPage` | Rename Master | Impress/Draw |
| `.uno:RenamePage` | Rename Page... | Impress/Draw |
| `.uno:RenameSlide` | Rename Slide... | Impress/Draw |
| `.uno:RenameSlideSection` | Rename Section... | Impress/Draw |
| `.uno:RenameTable` | Rename Sheet... | Calc |
| `.uno:Repair` | – | – |
| `.uno:ReplyComment` | Reply Comment | Alle |
| `.uno:ReplyToAnnotation` | – | – |
| `.uno:ReportIssue` | – | – |
| `.uno:RequestTextEdit` | – | – |
| `.uno:ResetAttributes` | Clear | Calc, Chart, Writer |
| `.uno:ResizeCalcTable` | Resize Table... | Calc |
| `.uno:ResolveComment` | Resolved | Alle |
| `.uno:ResolveCommentThread` | Resolved Thread | Alle |
| `.uno:ReturnTextEdit` | – | – |
| `.uno:RightPara` | Right | Alle |
| `.uno:RotateRight` | Rotate 90° Right | Writer |
| `.uno:RowColSelCount` | – | – |
| `.uno:RowHeight` | Row Height... | Calc |
| `.uno:RowOperations` | Row | Calc |
| `.uno:RulerChangeState` | – | – |
| `.uno:RunMacro` | Run Macro... | Alle |
| `.uno:SamplingDialog` | Sampling... | Calc |
| `.uno:Save` | Save | Alle |
| `.uno:SaveAs` | Save As... | Alle |
| `.uno:SaveGraphic` | Save Image | Alle, Impress/Draw |
| `.uno:SearchDialog` | Find and Replace... | Alle |
| `.uno:Sections` | – | – |
| `.uno:SecurityLabel` | Security Label... | Calc, Impress/Draw, Writer |
| `.uno:SelectAll` | Select All | Alle |
| `.uno:SelectBackground` | Set Background Image... | Impress/Draw |
| `.uno:SelectColumn` | Select Column | Calc |
| `.uno:SelectDB` | Select Range... | Calc |
| `.uno:SelectObject` | Select | Alle |
| `.uno:SelectRow` | Select Row | Calc |
| `.uno:SelectSheetView` | Select Sheet View | Calc |
| `.uno:SelectTable` | Select Table | Impress/Draw, Writer |
| `.uno:SelectTheme` | – | – |
| `.uno:SelectionMode` | Selection Mode | Writer |
| `.uno:SendToBack` | Send to Back | Alle |
| `.uno:SentenceCheckingPackages` | – | – |
| `.uno:SetBorderStyle` | Borders (Shift to overwrite) | Alle |
| `.uno:SetDefault` | Clear | Alle |
| `.uno:SetDefaultTableStyle` | – | – |
| `.uno:SetDocumentProperties` | Properties... | Alle |
| `.uno:SetHyperlink` | – | – |
| `.uno:SetLanguageAllTextMenu` | For All Text | Alle |
| `.uno:SetLanguageParagraphMenu` | For Paragraph | Alle |
| `.uno:SetLanguageSelectionMenu` | For Selection | Alle |
| `.uno:SetLongTopBottomMargin` | – | – |
| `.uno:SetMinimalColumnWidth` | Minimal Column Width | Impress/Draw, Writer |
| `.uno:SetMinimalRowHeight` | Minimal Row Height | Impress/Draw, Writer |
| `.uno:SetOptimalColumnWidth` | Optimal Column Width | Calc, Impress/Draw, Writer |
| `.uno:SetOptimalColumnWidthDirect` | Optimal Column Width, direct | Calc |
| `.uno:SetOptimalRowHeight` | Optimal Row Height | Calc, Impress/Draw, Writer |
| `.uno:SetOutline` | Outline Format | Alle |
| `.uno:SetTabBgColor` | Sheet Tab Color... | Calc |
| `.uno:SetTableStyle` | – | – |
| `.uno:Settings` | – | – |
| `.uno:Shadowed` | Shadow | Alle |
| `.uno:ShapesMenu` | Shape | Alle |
| `.uno:ShareDocument` | Share Spreadsheet... | Calc |
| `.uno:SheetGeometryData` | – | – |
| `.uno:SheetMenu` | Sheet | Calc |
| `.uno:SheetRightToLeft` | Sheet Right-To-Left | Calc |
| `.uno:Show` | Show Sheet... | Calc |
| `.uno:ShowAllNotes` | Show All Comments | Calc |
| `.uno:ShowAnnotations` | Show Comments | Alle |
| `.uno:ShowBoundaries` | Show Boundaries | Writer |
| `.uno:ShowChanges` | Show... | Calc |
| `.uno:ShowColumn` | Show Columns | Calc |
| `.uno:ShowDetail` | Show Details | Alle |
| `.uno:ShowLicense` | License Information | Alle |
| `.uno:ShowResolvedAnnotations` | Show resolved comments | Writer |
| `.uno:ShowRow` | Show Rows | Calc |
| `.uno:ShowRuler` | Rulers | Impress/Draw |
| `.uno:ShowSlide` | Show Slide | Impress/Draw |
| `.uno:ShowTrackedChanges` | Show Tracked Changes | Writer |
| `.uno:Shrink` | Decrease | Alle |
| `.uno:Shuffle` | Shuffle | Calc |
| `.uno:Sidebar` | Sidebar | Alle |
| `.uno:SidebarDeck` | – | – |
| `.uno:SidebarHide` | – | – |
| `.uno:SidebarShow` | – | – |
| `.uno:Signature` | Digital Signatures... | Alle |
| `.uno:Size` | Size | Alle |
| `.uno:SlideChangeWindow` | Slide Transition | Impress/Draw |
| `.uno:SlideMasterPage` | Master Slide | Impress/Draw |
| `.uno:SlideMenu` | Slide | Impress/Draw |
| `.uno:SlideSections` | – | – |
| `.uno:SlideSetup` | Format Slide | Impress/Draw |
| `.uno:SmallCaps` | Small capitals | Alle |
| `.uno:SolverDialog` | Solver... | Calc |
| `.uno:SortAscending` | Sort Ascending | Calc |
| `.uno:SortDescending` | Sort Descending | Calc |
| `.uno:SpacePara1` | Line Spacing: 1 | Alle |
| `.uno:SpacePara115` | Line Spacing: 1.15 | Alle |
| `.uno:SpacePara15` | Line Spacing: 1.5 | Alle |
| `.uno:SpacePara2` | Line Spacing: 2 | Alle |
| `.uno:Spacing` | Character Spacing | Alle |
| `.uno:SpellCheckIgnore` | Ignore | Alle |
| `.uno:SpellDialog` | Spelling... | Alle |
| `.uno:SpellOnline` | Auto Spellcheck | Alle |
| `.uno:SpellingAndGrammarDialog` | Spelling | Alle |
| `.uno:SplitCell` | Split Cells... | Alle, Calc |
| `.uno:SplitTable` | Split Table... | Writer |
| `.uno:StarShapes` | Star Shapes | Alle |
| `.uno:StartFormula` | – | – |
| `.uno:StartWithPresentation` | – | – |
| `.uno:StatePageNumber` | Page Number | Writer |
| `.uno:StateTableCell` | Cell | Alle |
| `.uno:StateWordCount` | – | – |
| `.uno:StatisticsMenu` | Statistics | Calc |
| `.uno:StatusBarFunc` | – | – |
| `.uno:StatusDocPos` | Position in Document | Calc |
| `.uno:StatusSelectionMode` | Selection Mode | Calc |
| `.uno:Strikeout` | Strikethrough | Alle |
| `.uno:StyleApply` | Apply Style | Alle, Writer |
| `.uno:StyleNewByExampleimg` | – | – |
| `.uno:StyleUpdateByExampleimg` | – | – |
| `.uno:SubScript` | Subscript | Alle, Writer |
| `.uno:SummarizeWithPivot` | Pivot Table... | Calc |
| `.uno:SuperScript` | Superscript | Alle, Writer |
| `.uno:SwitchControlDesignMode` | Toggle Design Mode | Alle |
| `.uno:SymbolShapes` | Symbol Shapes | Alle |
| `.uno:TTestDialog` | Paired t-test... | Calc |
| `.uno:TabBgColor` | Tab Color | Calc |
| `.uno:TableAutoFitMenu` | Size | Alle |
| `.uno:TableCellBackgroundColor` | Table Cell Background Color | Writer |
| `.uno:TableChangeCurrentBorderPosition` | – | – |
| `.uno:TableDeleteMenu` | Delete | Alle |
| `.uno:TableDialog` | Table Properties... | Impress/Draw, Writer |
| `.uno:TableInsertMenu` | Insert | Alle |
| `.uno:TableMenu` | Table | Alle |
| `.uno:TableNumberFormatDialog` | Number Format... | Writer |
| `.uno:TableSelectMenu` | Select | Writer |
| `.uno:TableSort` | Sort... | Impress/Draw, Writer |
| `.uno:TableStyle` | – | – |
| `.uno:TableStyleList` | – | – |
| `.uno:TableStyleSettings` | – | – |
| `.uno:TableStyles` | – | – |
| `.uno:TableTotalRow` | Totals Row | Calc |
| `.uno:TakeOverTextEdit` | – | – |
| `.uno:TaskPaneInsertPage` | Insert Slide | Impress/Draw |
| `.uno:Text` | Text Box | Alle |
| `.uno:TextAlign` | Align Text | Alle |
| `.uno:TextEditLocked` | – | – |
| `.uno:TextFormField` | Text Form Field | Alle |
| `.uno:TextFormFields` | – | – |
| `.uno:TextToColumns` | Text to Columns... | Calc |
| `.uno:TextWrap` | Text Wrap... | Writer |
| `.uno:ThemeDialog` | Theme... | Alle |
| `.uno:ThesaurusDialog` | Thesaurus... | Alle, Writer |
| `.uno:TitlePageDialog` | Title Page... | Writer |
| `.uno:ToggleGridHorizontal` | Horizontal Grids | Chart |
| `.uno:ToggleGridVertical` | Vertical Grids | Chart |
| `.uno:ToggleLegend` | Legend On/Off | Chart |
| `.uno:ToggleMergeCells` | Merge and Center Cells | Calc |
| `.uno:TogglePrintGrid` | Print Grid Lines | Calc |
| `.uno:ToggleRelative` | Cycle Cell Reference Types | Calc |
| `.uno:ToggleSheetGrid` | View Grid Lines | Calc |
| `.uno:ToolbarMode` | – | – |
| `.uno:ToolsMenu` | Tools | Alle |
| `.uno:TraceChangeMode` | Record | Calc |
| `.uno:TrackChanges` | Record | Writer |
| `.uno:TrackChangesInAllViews` | Record changes in all views | Writer |
| `.uno:TrackChangesInThisView` | Record changes in this view | Writer |
| `.uno:TransformDialog` | Position and Size... | Alle |
| `.uno:Translate` | Translate... | Writer |
| `.uno:TranslationProviders` | – | – |
| `.uno:Underline` | Underline | Alle |
| `.uno:UnderlineDouble` | Double Underline | Alle |
| `.uno:Undo` | Undo | Alle |
| `.uno:Ungroup` | Ungroup... | Alle |
| `.uno:UngroupSparklines` | Ungroup Sparklines | Calc |
| `.uno:UnsetCellsReadOnly` | Unprotect Cells | Writer |
| `.uno:UpSearch` | Find Previous  (Shift+F3) | Alle |
| `.uno:UpdateAll` | Update All | Writer |
| `.uno:UpdateBookmark` | – | – |
| `.uno:UpdateBookmarks` | – | – |
| `.uno:UpdateCurIndex` | Update Index | Writer |
| `.uno:UpdateField` | – | – |
| `.uno:UpdateFields` | – | – |
| `.uno:UpdateSections` | – | – |
| `.uno:UpdateTextFormField` | – | – |
| `.uno:Validation` | Validity... | Calc |
| `.uno:Vector` | – | – |
| `.uno:VectorPrimitives` | – | – |
| `.uno:VectorRenderingFont` | – | – |
| `.uno:VectorRenderingGraphics` | – | – |
| `.uno:VerticalText` | Vertical Text | Alle |
| `.uno:View3D` | 3D View... | Chart |
| `.uno:ViewAnnotations` | – | – |
| `.uno:ViewAnnotationsPosition` | – | – |
| `.uno:ViewMenu` | View | Alle |
| `.uno:ViewRowColumnHeaders` | View Headers | Calc |
| `.uno:Watermark` | Watermark... | Writer |
| `.uno:WidgetTestDialog` | – | – |
| `.uno:WordCountDialog` | Word Count... | Writer |
| `.uno:WrapAnchorOnly` | Wrap First Paragraph | Writer |
| `.uno:WrapContour` | Wrap Contour On | Writer |
| `.uno:WrapIdeal` | Optimal | Writer |
| `.uno:WrapLeft` | Before | Writer |
| `.uno:WrapMenu` | Wrap | Writer |
| `.uno:WrapOff` | None | Writer |
| `.uno:WrapOn` | Parallel | Writer |
| `.uno:WrapRight` | After | Writer |
| `.uno:WrapText` | Wrap Text | Calc |
| `.uno:WrapThrough` | Through | Writer |
| `.uno:WrapThroughTransparencyToggle` | In Background | Writer |
| `.uno:XLineColor` | Line Color | Alle |
| `.uno:XLineStyle` | Line Style | Alle |
| `.uno:ZTestDialog` | Z-test... | Calc |
| `.uno:Zoom` | Zoom... | Alle |
| `.uno:ZoomMinus` | Zoom Out | Alle |
| `.uno:ZoomPlus` | Zoom In | Alle |
| `.uno:invertbackground` | – | – |
| `.uno:showannotations` | – | – |
| `.uno:showruler` | – | – |
| `.uno:showstatusbar` | – | – |
