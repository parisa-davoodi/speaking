<template>
    <div
        style="max-height: 100%; height: 100%;"
        ref="container"
        v-bind:class="{
            'breakCursor' : mouseState === 'break',
            'regionCursor' : mouseState === 'region',
            'running-process' : modelData && modelData.__isWaiting,
        }"
    >
        <v-dialog
            v-model="timeSheet_dialog"
            scrollable
            max-width="max-content"
        >
            <template v-slot:activator="{ on, attrs }">
                <div v-bind="attrs" v-on="on" style="display: none !important;"></div>
            </template>
            <v-card>
                <div class="time-sheet-dialog">
                    <div class="time-sheet-dialog-header">Export Examiners Time Sheet</div>
                    <div class="time-sheet-dialog-body">
                        <div class="time-sheet-dialog-content">
                            <div class="time-sheet-dialog-content-main">
                                <div class="time-sheet-factor-title">From:</div>
                                <div class="time-sheet-dialog-content-main-data"><input v-model="timeSheetStartDate">
                                </div>
                                <div class="time-sheet-factor-title">To:</div>
                                <div class="time-sheet-dialog-content-main-data"><input v-model="timeSheetEndDate">
                                </div>
                                <div class="time-sheet-factor-title">Last Export:</div>
                                <div class="time-sheet-dialog-content-main-data time-sheet-previous-export-drop-area">
                                    <div
                                        class="time-sheet-previous-export-drop-area-preview-text"
                                        @dragover="timeSheetPreviousFileDragOver($event)"
                                        @drop="timeSheetPreviousFileDrop($event)"
                                    >
                                        <div v-if="!timeSheetPreviousFileName">drop the file here or&nbsp;<div
                                            class="link-button">open
                                        </div>&nbsp;it
                                        </div>
                                        <div v-else>{{ timeSheetPreviousFileName }}</div>
                                    </div>
                                </div>
                                <div class="time-sheet-factor-title" v-if="timeSheetMessage">Export Failed:</div>
                                <div class="time-sheet-dialog-content-main-data time-sheet-dialog-message"
                                     v-if="timeSheetMessage">{{ timeSheetMessage }}
                                </div>
                            </div>
                            <div class="time-sheet-dialog-content-shortcuts">
                                <div class="time-sheet-dialog-content-shortcuts-month"
                                     @click="selectTimeSheetMonth(-2)">
                                    {{ Date.today.nextMonthStart(-2).dateFormat("mmm") }}
                                </div>
                                <div class="time-sheet-dialog-content-shortcuts-month"
                                     @click="selectTimeSheetMonth(-1)">
                                    {{ Date.today.nextMonthStart(-1).dateFormat("mmm") }}
                                </div>
                                <div class="time-sheet-dialog-content-shortcuts-month" @click="selectTimeSheetMonth(0)">
                                    {{ Date.today.dateFormat("mmm") }}
                                </div>
                                <div class="time-sheet-dialog-content-shortcuts-month"
                                     @click="selectTimeSheetMonth(+1)">
                                    {{ Date.today.nextMonthStart(+1).dateFormat("mmm") }}
                                </div>
                            </div>
                        </div>
                        <div class="time-sheet-dialog-content-progress"
                             v-if="timeSheetProgressPercentage !== null && timeSheetProgressPercentage !== undefined && timeSheetProgressPercentage < 1">
                            <div class="time-sheet-dialog-content-progress-desc">
                                <div class="time-sheet-dialog-content-progress-desc-title">
                                    {{ timeSheetProgressDescTitle }}
                                </div>
                                <div class="time-sheet-dialog-content-progress-desc-scale">
                                    {{ timeSheetProgressDescScale }}
                                </div>
                            </div>
                            <div class="time-sheet-dialog-content-progress-bar">
                                <div class="time-sheet-dialog-content-progress-bar-done"
                                     :style="`width: ${timeSheetProgressPercentage * 100}%;`">
                                </div>
                            </div>
                        </div>
                        <div class="time-sheet-dialog-tools">
                            <div class="time-sheet-dialog-buttons">
                                <button class="time-sheet-dialog-cancel-button" @click="onTimeSheetCancel()">
                                    Cancel
                                </button>
                                <button
                                    :disabled="!timeSheetDownloadCanBeStarted()"
                                    class="time-sheet-dialog-ok-button"
                                    v-bind:class="{'disable-style': !this.timeSheetDownloadCanBeStarted()}"
                                    @click="onTimeSheetOk()"
                                >
                                    OK
                                </button>
                            </div>
                        </div>
                    </div>
                </div>
            </v-card>
        </v-dialog>
        <div v-bind:class="{'hide': !mini}" v-if="modelData" class="miniSide" @click="_stopPropagation($event)">
            <div class="tools-folder">
                <div class="d-block">
                    <div class="d-inline-block symbol-containor">
                        <v-icon class="symbol waiting-symbol">mdi-progress-alert</v-icon>
                    </div>
                </div>
                <div class="factor">
                    <div class="d-inline-block symbol-containor"
                         v-bind:class="{'disabledStyl' : !modelData.__candidateSearchFilters.__any}">
                        <v-icon class="symbol" style="color: #1d643b">
                            mdi-search-web
                        </v-icon>
                    </div>
                    <div v-if="modelData.__board" class="d-inline-block float-right"
                         style="text-decoration: underline; color: #e34848;">
                        <v-menu
                            v-if="modelData.__board && modelData.__board.searchCandidate.__any && modelData.__board.searchCandidate.__resultCount !== 0"
                            close-on-click
                            :close-on-content-click="false"
                            offset-x
                            class="tooltip-panel"
                        >
                            <template v-slot:activator="{ on, attrs }">
                                <div class="d-inline-block showCount"
                                     v-bind="attrs"
                                     v-on="on"
                                     @click="setActiveTooltipKey($event, 'mini-search')"
                                >
                                    {{ modelData.__board.searchCandidate.__resultCount }}
                                </div>
                            </template>
                            <CandidateTooltip
                                :model="activeTooltipKey === 'mini-search' ? modelData.__board.searchCandidate.tooltipModel() : null"
                                :showMultiple="true"/>
                        </v-menu>
                        <div v-else-if="modelData.__board && modelData.__board.searchCandidate.__any"
                             class="d-inline-block showCount-0">
                            -
                        </div>
                    </div>
                </div>
                <div class="factor" v-if="modelData && modelData.__messages">
                    <div class="d-inline-block symbol-containor">
                        <v-icon class="symbol" style="color: red">
                            mdi-alert-circle
                        </v-icon>
                    </div>
                    <div class="d-inline-block float-right"
                         style="text-decoration: underline; color: #e34848;">
                        <v-dialog
                            v-model="miniMsg_dialog"
                            v-if="modelData.__messages.count !== 0"
                            scrollable
                            max-width="max-content"
                        >
                            <template v-slot:activator="{ on, attrs }">
                                <div class="d-inline-block showCount"
                                     v-bind="attrs"
                                     v-on="on">
                                    {{ modelData.__messages.count }}
                                </div>
                            </template>
                            <div class="msg_container">
                                <div class="msg_header">messages</div>
                                <div>
                                    <div v-for="(msgGp,i) in modelData.__messages.folders"
                                         data-isOpen="f">
                                        <div style="padding-top: 10px;">
                                            <v-icon @click="msgGp.isOpen = !msgGp.isOpen"
                                                    v-bind:class="{'menu_down_icon_open': msgGp.isOpen}"
                                                    class="menu_down_icon">mdi-menu-down
                                            </v-icon>
                                            <div class="msg_Title">{{ msgGp.title }}</div>
                                        </div>
                                        <div v-if="msgGp.isOpen">
                                            <div v-for="(child,i) in msgGp.messages"
                                                 class="subMsg_Title">
                                                {{ child.text }}
                                            </div>
                                        </div>
                                    </div>
                                </div>
                                <div>
                                    <v-btn
                                        color="blue darken-1"
                                        text
                                        @click="miniMsg_dialog = false"
                                    >
                                        Close
                                    </v-btn>
                                    <v-btn
                                        color="blue darken-1"
                                        text
                                        @click="miniMsg_dialog = false"
                                    >
                                        Save
                                    </v-btn>
                                </div>
                            </div>
                        </v-dialog>
                        <div v-else-if="modelData.__messages.count === 0"
                             class="d-inline-block showCount-0">
                            -
                        </div>
                    </div>
                </div>
                <div class="d-inline-block selectable-button"
                     @click="onUseTextStateClick($event)"
                     v-bind:class="{'disabledStyl': modelData.__candidateSearchFilters.__useText === 'ignore'}"
                >
                    <v-icon>
                        mdi-format-text
                    </v-icon>
                </div>
                <div class="d-inline-block selectable-button"
                     @click="onRegionalityStateClick($event)"
                     v-if="modelData.__candidateSearchFilters.__regionality === 'not-regional'"
                     v-bind:class="{'disabledStyl': modelData.__candidateSearchFilters.__regionality === 'ignore'}"
                >
                    <v-icon>
                        mdi-map-marker-off
                    </v-icon>
                </div>
                <div class="d-inline-block selectable-button"
                     @click="onRegionalityStateClick($event)"
                     v-if="modelData.__candidateSearchFilters.__regionality === 'regional' || modelData.__candidateSearchFilters.__regionality === 'ignore'"
                     v-bind:class="{'disabledStyl': modelData.__candidateSearchFilters.__regionality === 'ignore'}"
                >
                    <v-icon>
                        mdi-map-marker-radius
                    </v-icon>
                </div>
                <div class="d-inline-block selectable-button"
                     @click="onNoteStateClick($event)"
                     v-bind:class="{'disabledStyl': modelData.__candidateSearchFilters.__note === 'ignore'}"
                >
                    <v-icon>
                        mdi-note-text
                    </v-icon>
                </div>
                <div class="d-inline-block selectable-button"
                     @click="onEditStateClick($event)"
                     v-bind:class="{'disabledStyl': modelData.__candidateSearchFilters.__edit === 'ignore'}"
                >
                    <v-icon>
                        mdi-move-resize-variant
                    </v-icon>
                </div>
            </div>
            <div class="tools-folder mouse-pointer-folder">
                <div class="d-block selectable-button"
                     @click="onDefaultMouseClick()"
                     v-bind:class="{'disabledStyl': mouseState !== 'default'}"
                >
                    <v-icon>
                        mdi-cursor-default-outline
                    </v-icon>
                </div>
                <div class="d-inline-block selectable-button"
                     @click="onBreakMouseClick($event)"
                     v-bind:class="{'disabledStyl': mouseState !== 'break'}"
                >
                    <v-icon>
                        mdi-bell-sleep
                    </v-icon>
                </div>
                <div class="d-inline-block selectable-button"
                     @click="onRegionMouseClick($event)"
                     v-bind:class="{'disabledStyl': mouseState !== 'region'}"
                >
                    <v-icon>
                        mdi-map-marker-plus
                    </v-icon>
                </div>
            </div>
            <div class="tools-folder" v-bind:class="{'invisible' : !modelData.__anyVisibilityForce}">
                <div class="d-inline-block command-button"
                     @click="onVisibilityForceClick()"
                >
                    <v-icon>
                        mdi-account-remove
                    </v-icon>
                </div>
            </div>

            <div v-if="modelData.__targetExam" class="targetExamSideBarContainer">
                <div class="targetExamSideBar" @click="onSideBarTargetExamClick()">
                    {{ modelData.__targetExam.title }}
                </div>
            </div>
            <div class="saveBtnContainer">
                <button
                    color="success"
                    class="saveBtn"
                    @click="onSaveClick"
                    v-bind:class="{'disableSaveBtn': !modelData.saveButton.__enabled}"
                >
                    Save
                </button>
            </div>
        </div>
        <div v-bind:class="{'hide': mini}" v-if="modelData" class="maxSide" @click="_stopPropagation($event)">
            <div class="d-block" style="padding: 6px;">
                <div class="d-block">
                    <div class="d-inline-block symbol-containor">
                        <v-icon class="symbol waiting-symbol">mdi-progress-alert</v-icon>
                    </div>
                </div>
            </div>
            <div v-if="modelData && modelData.__messages" class="d-block" style="padding: 6px;">
                <div class="d-inline-block symbol-containor">
                    <v-icon class="symbol" style="color: red">mdi-alert-circle</v-icon>
                </div>
                <div class="d-inline-block float-right"
                     style="text-decoration: underline; color: red;">
                    <v-dialog
                        v-model="miniMsg_dialog"
                        v-if="modelData.__messages.count !== 0"
                        scrollable
                        max-width="max-content"
                    >
                        <template v-slot:activator="{ on, attrs }">
                            <div class="d-inline-block showCount"
                                 v-bind="attrs"
                                 v-on="on">
                                {{ modelData.__messages.count }}
                            </div>
                        </template>
                        <div class="msg_container">
                            <div class="msg_header">messages</div>
                            <div>
                                <div v-for="(msgGp,i) in modelData.__messages.folders"
                                     data-isOpen="f">
                                    <div style="padding-top: 10px;">
                                        <v-icon @click="msgGp.isOpen = !msgGp.isOpen"
                                                v-bind:class="{'menu_down_icon_open': msgGp.isOpen}"
                                                class="menu_down_icon">mdi-menu-down
                                        </v-icon>
                                        <div class="msg_Title">{{ msgGp.title }}</div>
                                    </div>
                                    <div v-if="msgGp.isOpen">
                                        <div v-for="(child,i) in msgGp.messages"
                                             class="subMsg_Title">
                                            {{ child.text }}
                                        </div>
                                    </div>
                                </div>
                            </div>
                            <div>
                                <v-btn
                                    color="blue darken-1"
                                    text
                                    @click="miniMsg_dialog = false"
                                >
                                    Close
                                </v-btn>
                                <v-btn
                                    color="blue darken-1"
                                    text
                                    @click="miniMsg_dialog = false"
                                >
                                    Save
                                </v-btn>
                            </div>
                        </div>
                    </v-dialog>
                    <div v-else-if="modelData.__messages.count == 0"
                         class="d-inline-block showCount-0">
                        -
                    </div>
                </div>
            </div>
            <div class="tools-folder">
                <input class="searchCandidateInp" placeholder="search candidate..." v-model="searchInpTxt"
                       @input="onSearchCndtInput($event)"/>
            </div>
            <div class="tools-folder search-options">
                <div class="search-buttons">
                    <div class="d-inline-block selectable-button"
                         @click="onUseTextStateClick($event)"
                         v-bind:class="{'disabledStyl': modelData.__candidateSearchFilters.__useText === 'ignore'}"
                    >
                        <v-icon>
                            mdi-format-text
                        </v-icon>
                    </div>
                    <div class="d-inline-block selectable-button"
                         @click="onRegionalityStateClick($event)"
                         v-if="modelData.__candidateSearchFilters.__regionality === 'not-regional'"
                         v-bind:class="{'disabledStyl': modelData.__candidateSearchFilters.__regionality === 'ignore'}"
                    >
                        <v-icon>
                            mdi-map-marker-off
                        </v-icon>
                    </div>
                    <div class="d-inline-block selectable-button"
                         @click="onRegionalityStateClick($event)"
                         v-if="modelData.__candidateSearchFilters.__regionality === 'regional' || modelData.__candidateSearchFilters.__regionality === 'ignore'"
                         v-bind:class="{'disabledStyl': modelData.__candidateSearchFilters.__regionality === 'ignore'}"
                    >
                        <v-icon>
                            mdi-map-marker-radius
                        </v-icon>
                    </div>
                    <div class="d-inline-block selectable-button"
                         @click="onNoteStateClick($event)"
                         v-bind:class="{'disabledStyl': modelData.__candidateSearchFilters.__note === 'ignore'}"
                    >
                        <v-icon>
                            mdi-note-text
                        </v-icon>
                    </div>
                    <div class="d-inline-block selectable-button"
                         @click="onEditStateClick($event)"
                         v-bind:class="{'disabledStyl': modelData.__candidateSearchFilters.__edit === 'ignore'}"
                    >
                        <v-icon>
                            mdi-move-resize-variant
                        </v-icon>
                    </div>
                </div>
                <div>
                    <v-menu
                        v-if="modelData.__board && modelData.__board.searchCandidate.__any && modelData.__board.searchCandidate.__resultCount !== 0"
                        close-on-click
                        :close-on-content-click="false"
                        offset-x
                        class="tooltip-panel"
                    >
                        <template v-slot:activator="{ on, attrs }">
                            <div class="showCount search-result-long"
                                 v-bind="attrs"
                                 v-on="on"
                                 @click="setActiveTooltipKey($event, 'maxi-search')"
                            >
                                {{ modelData.__board.searchCandidate.__resultCount }} item(s) found ...
                            </div>
                        </template>
                        <CandidateTooltip
                            :model="activeTooltipKey === 'maxi-search' ? modelData.__board.searchCandidate.tooltipModel() : null"
                            :showMultiple="true"/>
                    </v-menu>
                    <div v-else-if="modelData.__board && modelData.__board.searchCandidate.__any"
                         class="showCount-0 search-result-long">
                        no items found
                    </div>
                </div>
            </div>
            <div class="tools-folder mouse-pointer-folder">
                <div class="d-inline-block selectable-button"
                     @click="onDefaultMouseClick()"
                     v-bind:class="{'disabledStyl': mouseState !== 'default'}"
                >
                    <v-icon>
                        mdi-cursor-default-outline
                    </v-icon>
                </div>
                <div class="d-inline-block selectable-button"
                     @click="onBreakMouseClick($event)"
                     v-bind:class="{'disabledStyl': mouseState !== 'break'}"
                >
                    <v-icon>
                        mdi-bell-sleep
                    </v-icon>
                </div>
                <div class="d-inline-block selectable-button"
                     @click="onRegionMouseClick($event)"
                     v-bind:class="{'disabledStyl': mouseState !== 'region'}"
                >
                    <v-icon>
                        mdi-map-marker-plus
                    </v-icon>
                </div>
            </div>
            <div class="tools-folder list-buttons-folder">
                <div
                    class="d-inline-block selectable-button"
                    @click="onToggleShowExamsClick()"
                    v-bind:class="{'disabledStyl': !showExamsOnSidebar}">
                    <v-icon>
                        mdi-image-filter-none
                    </v-icon>
                </div>
                <div
                    class="d-inline-block selectable-button"
                    @click="onToggleShowExaminersClick()"
                    v-bind:class="{'disabledStyl': !showExaminersOnSidebar}">
                    <v-icon>
                        mdi-account-check
                    </v-icon>
                </div>


                <div
                    class="d-inline-block selectable-button"
                    @click="onTimeSheetClick()">
                    <v-icon>
                        mdi-account-clock
                    </v-icon>
                </div>


                <div class="d-inline-block command-button"
                     @click="onVisibilityForceClick()"
                     v-if="modelData.__anyVisibilityForce">
                    <v-icon>
                        mdi-account-remove
                    </v-icon>
                </div>
            </div>
            <div class="scrollSideBar">
                <div
                    class="fetch-more-button"
                    v-if="showExamsOnSidebar"
                    @click="fetchTheExamsBefore()"
                >
                    more exams ...
                </div>
                <div v-for="(exm, x) in modelData.__examAccessList" v-if="showExamsOnSidebar">
                    <div
                        @click="onExamSelectionClick(exm)"
                        class="examList"
                        v-bind:class="{
                            'currentExamHighlight' : modelData.__targetExam === exm,
                            'new-date-exam' : x > 0 && modelData.__examAccessList[x - 1].testSession.date.getTime() !== exm.testSession.date.getTime(),
                            'mock-exam' : exm.isMock,
                        }">
                        <div class="exam-list-color" :style="`background-color: ${exm.color};`"></div>
                        <div class="exam-list-title">
                            {{ exm.title }}
                            <span class="exam-list-title-ir">({{ exm.testSession.date.dateFormat("i-mm/i-dd") }})</span>
                        </div>
                    </div>
                </div>
                <div
                    class="fetch-more-button"
                    v-if="showExamsOnSidebar"
                    @click="fetchTheExamsAfter()"
                >
                    more exams ...
                </div>
                <div v-for="(_exmnr, x) in modelData.__allExaminers" v-if="showExaminersOnSidebar">
                    <div
                        @click="onExaminerSelectionClick($event, _exmnr.entity)"
                        class="examList"
                        v-bind:class="{'currentExamHighlight' : _exmnr.__visibility === 'force'}">
                        {{ _exmnr.entity.title }}
                    </div>
                </div>
            </div>
            <div class="saveBtnContainer">
                <button
                    color="success"
                    class="saveBtn"
                    @click="onSaveClick"
                    v-bind:class="{'disableSaveBtn': !modelData.saveButton.__enabled}"
                >
                    Save
                </button>
            </div>
        </div>
        <v-dialog
            v-model="examiner_dialog"
            scrollable
            max-width="max-content"
            persistent
            class="eft_dialog"
        >
            <template v-slot:activator="{ on, attrs }">
                <div v-bind="attrs" v-on="on" style="display: none !important;"></div>
            </template>
            <div class="eft_container" v-if="eftModel">
                <div class="eft-content">
                    <div>
                        <div class="eft-dialog-title">
                            <div class="eft-dialog-title-examiner">{{ eftModel.examinerTitle }}</div>
                            <div class="eft-dialog-title-range">{{ eftModel.__fromDateTitle }} -
                                {{ eftModel.__toDateTitle }}
                            </div>
                        </div>
                        <div class="eft-table-container">
                            <table>
                                <tr>
                                    <td></td>
                                    <td></td>
                                    <td @click="eftModel.fetchTheDaysBefore(10)">
                                        <div class="fetch-more-button">more days ...</div>
                                    </td>
                                    <td></td>
                                </tr>
                                <tr v-for="(dateElement, i) in eftModel.__dateElements"
                                    @click="eftModel.selectDateElementAt(i)"
                                    class="eft-date"
                                    v-bind:class="{ 'eft-date-selected' : dateElement.__isSelected }"
                                >
                                    <td>
                                        <div>{{ dateElement.weekDayTitle }}</div>
                                    </td>
                                    <td>
                                        <div>{{ dateElement.dateTitle }}</div>
                                    </td>
                                    <td class="eft-ranges">
                                        <div>
                                            <div v-for="(rangeElement, i) in dateElement.__rangeElements"
                                                 :style="`width: ${rangeElement.weight * 100}%`"
                                                 class="eft-range"
                                                 v-bind:class="{
                                                 'eft-range-direct' : rangeElement.theme === 'direct',
                                                 'eft-range-template' : rangeElement.theme === 'template',
                                                 'eft-range-empty' : rangeElement.theme === 'empty',
                                                 'eft-range-unknown' : rangeElement.theme === 'unknown',
                                             }"
                                            >
                                                {{ rangeElement.title }}
                                            </div>
                                        </div>
                                    </td>
                                    <td class="eft-date-status">
                                        <div>
                                            <v-icon
                                                v-bind:class="{
                                            'eft-date-error' : dateElement.__status === 'error',
                                            'eft-date-normal' : dateElement.__status === 'normal',
                                            'eft-date-warning' : dateElement.__status === 'warning',
                                        }"
                                            >
                                                {{
                                                    dateElement.__status === 'locked' ? 'mdi-lock-outline' : dateElement.__status === 'warning' ? 'mdi-alert-circle-outline' : dateElement.__status === 'error' ? 'mdi-alert' : ''
                                                }}
                                            </v-icon>
                                        </div>
                                    </td>
                                </tr>
                                <tr>
                                    <td></td>
                                    <td></td>
                                    <td @click="eftModel.fetchTheDaysAfter(10)">
                                        <div class="fetch-more-button">more days ...</div>
                                    </td>
                                    <td></td>
                                </tr>
                            </table>
                        </div>
                    </div>
                    <div class="eft-control-container">
                        <div v-if="eftModel.selectedDateFreeTimesElement.__isVisible">
                            <div class="eft-control-title">{{ eftModel.selectedDateFreeTimesElement.__title }}</div>
                            <div class="eft-control-values">
                                <div class="eft-control-ranges">
                                    <input :disabled="!eftModel.selectedDateFreeTimesElement.__isEditable"
                                           placeholder="from ..." v-model="eftDirectStart"/>
                                    <input :disabled="!eftModel.selectedDateFreeTimesElement.__isEditable"
                                           placeholder="to ..." v-model="eftDirectEnd"/>
                                    <input :disabled="!eftModel.selectedDateFreeTimesElement.__isEditable"
                                           placeholder="from ..." v-model="eftDirectStart2"/>
                                    <input :disabled="!eftModel.selectedDateFreeTimesElement.__isEditable"
                                           placeholder="to ..." v-model="eftDirectEnd2"/>
                                </div>
                                <div class="eft-control-commands">
                                    <div v-if="eftModel.selectedDateFreeTimesElement.__isEditable"
                                         @click="onEftSetFull()">full
                                    </div>
                                    <div v-if="eftModel.selectedDateFreeTimesElement.__isEditable"
                                         @click="onEftSetOff()">off
                                    </div>
                                    <div v-if="eftModel.selectedDateFreeTimesElement.__isEditable"
                                         @click="onEftSetUnknown()"
                                         class="rare-link"
                                    >unk
                                    </div>
                                </div>
                            </div>
                            <div class="eft-control-constraints">
                                <div
                                    v-bind:class="{
                                        'disable-style': eftConstraintIsDisable('vcs'),
                                        'none-proposed-style': eftOptionIsNoneProposed('vcs'),
                                    }"
                                >
                                    <input v-model="eftAllowVCS" type="checkbox"
                                           :disabled="eftConstraintIsDisable('vcs')">
                                    <div>VCS</div>
                                </div>
                                <!--                                <div-->
                                <!--                                    v-bind:class="{-->
                                <!--                                        'disable-style': eftConstraintIsDisable('paperBaseUKVI'),-->
                                <!--                                        'none-proposed-style': eftOptionIsNoneProposed('paperBaseUKVI'),-->
                                <!--                                    }"-->
                                <!--                                >-->
                                <!--                                    <input v-model="eftAllowPaperBaseUKVI" type="checkbox"-->
                                <!--                                           :disabled="eftConstraintIsDisable('paperBaseUKVI')">-->
                                <!--                                    <div>Paper Base UKVI</div>-->
                                <!--                                </div>-->
                                <div
                                    v-bind:class="{
                                        'disable-style': eftConstraintIsDisable('residenceFTF'),
                                        'none-proposed-style': eftOptionIsNoneProposed('residenceFTF'),
                                    }"
                                >
                                    <input v-model="eftAllowResidenceFTF" type="checkbox"
                                           :disabled="eftConstraintIsDisable('residenceFTF')">
                                    <div>Residence Face to Face</div>
                                </div>
                                <div
                                    v-bind:class="{
                                        'disable-style': eftConstraintIsDisable('otherCitiesFTF'),
                                        'none-proposed-style': eftOptionIsNoneProposed('otherCitiesFTF'),
                                    }"
                                >
                                    <input v-model="eftAllowOtherCitiesFTF" type="checkbox"
                                           :disabled="eftConstraintIsDisable('otherCitiesFTF')">
                                    <div>Other Cities Face to Face</div>
                                </div>
                            </div>
                            <div class="eft-control-note">
                                <input :disabled="!eftModel.selectedDateFreeTimesElement.__isEditable"
                                       placeholder="note ..." v-model="eftNote">
                            </div>
                            <div class="eft-control-note eft-control-admin-note">
                                <input placeholder="admin note ..." v-model="eftAdminNote">
                            </div>
                        </div>
                    </div>
                </div>
                <div class="eft-dialog-buttons">
                    <div @click="eft_dialog_cancel()" class="button-div eft-cancel-button">Cancel</div>
                    <div
                        @click="eft_dialog_ok()"
                        class="button-div eft-ok-button"
                        v-bind:class="{
                            'eft-ok-button-disabled' : !eftModel.submitElement.isEnable
                        }"
                    >
                        Ok
                    </div>
                </div>
            </div>
        </v-dialog>
        <div class="copyright-style">
            <div>cheshmeh v01.00 - © 2580 ifi</div>
        </div>
        <div
            class="mx-10 mu-10"
            ref="tableContainer"
            @mouseover="onTableContainerMouseOver($event)"
            v-if="modelData && modelData.__board">
            <table
                class="fixed_header my-table"
                @mousemove="tableMouseMove($event)"
                @mouseleave="onTableContainerMouseLeave()"
                ref="table"
            >
                <thead>
                <tr class="headerBorderBottom">
                    <th></th>
                    <th v-bind:class="{ 'highlight': modelData.__board.unplanned.__highlight, 'inactive': !modelData.__board.unplanned.__highlight }">
                        <div class="unplannedContainer">
                            <div
                                class="auto-plan"
                                v-if="modelData.__board.unplanned.__count !== 0"
                            >
                                <div @click="autoPlan()">auto plan</div>
                            </div>
                            <div>
                                <div class="d-inline-block float-left unplannedTxt">
                                    Unplanned
                                </div>
                                <v-menu
                                    v-if="modelData.__board.unplanned.__count !== 0"
                                    close-on-click
                                    :close-on-content-click="false"
                                    offset-x
                                    class="tooltip-panel"
                                >
                                    <template v-slot:activator="{ on, attrs }">
                                        <div
                                            class="d-inline-block showCount"
                                            v-bind="attrs"
                                            v-on="on"
                                            v-bind:class="{ 'outlineHighlight': modelData.__board.unplanned.__searchHighlight }"
                                            @dragstart="modelData.__board.unplanned.__count ? onDragStart($event,modelData.__board.unplanned.__unplannedTargetRegisters[0],'unplanned') : ''"
                                            @dragend="modelData.__board.unplanned.__count ? onDragEnd($event) : ''"
                                            @drag="modelData.__board.unplanned.__count ? onDrag($event) : ''"
                                            :draggable="modelData.__board.unplanned.__count ? true : false"
                                            @click="setActiveTooltipKey($event, 'unplanned')"
                                        >
                                            {{ modelData.__board.unplanned.__count }}
                                        </div>
                                    </template>
                                    <CandidateTooltip
                                        :model="activeTooltipKey === 'unplanned' ? modelData.__board.unplanned.tooltipModel(): null"/>
                                </v-menu>
                                <div
                                    class="d-inline-block showCount"
                                    v-else
                                    v-bind:class="{ 'outlineHighlight': modelData.__board.unplanned.__searchHighlight }"
                                >
                                    {{ modelData.__board.unplanned.__count }}
                                </div>
                            </div>
                        </div>
                    </th>
                    <th class="c-time-cell-header" v-for="(timeLine, i) in modelData.timeLines" :key="i" ref="rangeTd">
                        <div v-if="i < modelData.timeLines.length - 1" class="out"
                             v-bind:class="{ 'show-range' : hoveredTdStartIndex <= i && i < hoveredTdEndIndex }">
                            <div class="ins" v-bind:class="{'h-rotate' : timeLine.caption.length > 2}"
                                 v-html="timeLine.caption"></div>
                        </div>
                        <div v-if="i === modelData.timeLines.length - 1" class="out">
                            <div class="ins" v-bind:class="{'h-rotate' : timeLine.caption.length > 2}"
                                 v-html="timeLine.caption"></div>
                        </div>
                    </th>
                    <th class="h-warnings">
                        <div>
                            <div v-for="(wrnng, w) in modelData.__board.warnings"
                                 v-if="wrnng.issueCount !== 0">
                                <div
                                    v-if="wrnng.detailsType !== 'candidate-register'"
                                    class="warningTxt"
                                    v-bind:class="{
                                        'wt-error' : wrnng.severity === 'error',
                                        'wt-warning' : wrnng.severity === 'warning',
                                        'wt-hint' : wrnng.severity === 'hint',
                                        'wt-inform' : wrnng.severity === 'inform',
                                    }"
                                >
                                    {{ wrnng.title }}
                                </div>
                                <v-menu
                                    v-if="wrnng.detailsType === 'candidate-register'"
                                    close-on-click
                                    :close-on-content-click="false"
                                    class="tooltip-panel"
                                    offset-x
                                    bottom
                                    left
                                >
                                    <template v-slot:activator="{ on, attrs }">
                                        <div
                                            class="warningTxt w-detail"
                                            v-bind="attrs"
                                            v-on="on"
                                            @click="setActiveTooltipKey($event, wrnng)"
                                            v-bind:class="{
                                                'wt-error' : wrnng.severity === 'error',
                                                'wt-warning' : wrnng.severity === 'warning',
                                                'wt-hint' : wrnng.severity === 'hint',
                                                'wt-inform' : wrnng.severity === 'inform',
                                            }"
                                        >
                                            {{ wrnng.title }}
                                        </div>
                                    </template>
                                    <CandidateTooltip
                                        :model="activeTooltipKey === wrnng ? wrnng.tooltipModel() : null"/>
                                </v-menu>
                            </div>
                        </div>
                    </th>
                </tr>
                </thead>
                <tbody v-for="(dateLane, j) in modelData.__board.dateLanes" :key="j" style="display: contents"
                       v-bind:class="{
                        'exam-bg': dateLane.dateTitle.__color == 'exam',
                        'highLight-bg': dateLane.dateTitle.__color == 'highlight',
                        'warning-bg': dateLane.dateTitle.__color == 'warning',
                        'inactive-bg': dateLane.dateTitle.__color == 'inactive',
                    }">
                <tr class="first-space-row" v-bind:class="{'non-top-first-space-row' : j != 0}">
                    <td class="c-date-title" :rowspan="calculateWarningClspn(dateLane)">
                        <div class="c-date-title-content">
                            <div>{{ dateLane.date.dateFormat("mmm dd") }}</div>
                            <div>{{ dateLane.date.dateFormat("www") }}</div>
                            <div>({{ dateLane.date.dateFormat("i-mm/i-dd") }})</div>
                        </div>
                        <div
                            v-if="modelData.__targetExam.__mainStructureIsLoaded"
                            class="c-date-est"
                            v-bind:class="{
                                'c-date-est-any': dateLane.__expectedFTF || dateLane.__expectedVCS
                            }"
                        >
                            <div v-bind:class="{'est-off': !dateLane.__expectedFTF}"
                                 @click="toggleDateLaneExpectedSpeakingFTF(dateLane)">F
                            </div>
                            <div v-bind:class="{'est-off': !dateLane.__expectedVCS}"
                                 @click="toggleDateLaneExpectedSpeakingVCS(dateLane)">V
                            </div>
                        </div>
                    </td>
                    <td class="c-lane-title"></td>
                    <td class="c-time-cell" v-for="(itemn, n) in modelData.timeLines"
                        @mouseup="onMouseUp($event)" :data-time-index="n"></td>
                    <td class="c-warnings" :rowspan="calculateWarningClspn(dateLane)">
                        <div>
                            <div v-for="(wrnng, w) in dateLane.warnings"
                                 v-if="wrnng.issueCount !== 0">
                                <div
                                    v-if="wrnng.detailsType !== 'candidate-register'"
                                    class="warningTxt"
                                    v-bind:class="{
                                        'wt-error' : wrnng.severity === 'error',
                                        'wt-warning' : wrnng.severity === 'warning',
                                        'wt-hint' : wrnng.severity === 'hint',
                                        'wt-inform' : wrnng.severity === 'inform',
                                    }"
                                >
                                    {{ wrnng.title }}
                                </div>
                                <v-menu
                                    v-if="wrnng.detailsType === 'candidate-register'"
                                    close-on-click
                                    :close-on-content-click="false"
                                    class="tooltip-panel"
                                    offset-x
                                    bottom
                                    left
                                >
                                    <template v-slot:activator="{ on, attrs }">
                                        <div
                                            v-bind="attrs"
                                            v-on="on"
                                            class="warningTxt w-detail"
                                            @click="setActiveTooltipKey($event, wrnng)"
                                            v-bind:class="{
                                                'wt-error' : wrnng.severity === 'error',
                                                'wt-warning' : wrnng.severity === 'warning',
                                                'wt-hint' : wrnng.severity === 'hint',
                                                'wt-inform' : wrnng.severity === 'inform',
                                            }"
                                        >
                                            {{ wrnng.title }}
                                        </div>
                                    </template>
                                    <CandidateTooltip
                                        :model="activeTooltipKey === wrnng ? wrnng.tooltipModel() : null"/>
                                </v-menu>
                            </div>
                        </div>
                    </td>
                </tr>

                <tbody v-for="(examinrLane, ee) in dateLane.examinerLanes" :key="'X' + ee" style="display: contents"
                       v-if="examinrLane.__visible">
                <tr class="examiner-row">
                    <td class="c-lane-title examiner-title">
                        <div
                            v-bind:class="{'examinerTitle-containor': examinrLane.__bgColor === 'force'}"
                            @click="onExaminerTitleClick(examinrLane.examiner, dateLane.date)"
                        >
                            {{ examinrLane.examiner.title }}
                        </div>
                    </td>
                    <td class="c-time-cell" v-for="(itemn, n) in examinrLane.__timeCells" :key="'Z' + n"
                        v-bind:class="{
                                        'start-occupied-time-cell': examinrLane.range.__start === itemn.startTime,
                                        'end-occupied-time-cell': examinrLane.range.__end === itemn.endTime,
                                        'occupied-time-cell': itemn.__type !== 'external',
                                        'occupied-session': itemn.__type !== 'external' && itemn.__type !== 'break',
                                        't-vcs': itemn.__type === 'vcs',
                                        'bgType-none': itemn.__bgType === 'none',
                                        'bgType-free': itemn.__bgType === 'free',
                                        'bgType-notFree': itemn.__bgType === 'not-free',
                                        'bgType-otherExam': itemn.__bgType === 'other-exam',
                                        'bgType-break': itemn.__bgType === 'break',
                                        'bgType-thisExam': itemn.__bgType === 'this-exam',
                                        'rg-regional': itemn.__ispublic === false,
                                        'rg-non-regional': itemn.__ispublic === true,
                                        'time-cell-issue': itemn.multiPlan,
                                     }"
                        @mouseup="onMouseUp($event)"
                        @drop="onDrop($event,examinrLane,itemn)"
                        @dragover="onDragOver($event)"
                        @click="examinrLane.range.__start <= n && n < examinrLane.range.__end ? onExaminerRangeClick($event,examinrLane,itemn) : ''"
                        :colspan="itemn.timeCount"
                        :data-span-start="itemn.startTime"
                        :data-span-end="itemn.endTime"
                        :data-time-index="n"
                    >
                        <div>
                            <div v-if="examinrLane.range.__start === itemn.startTime" class="drawer start-drawer"
                                 @mousedown="onDrawerMouseDown($event,'start',dateLane, examinrLane )"
                                 @click="onDrawerClick($event)"> {{ drawerText }}
                            </div>
                            <div v-if="examinrLane.range.__end === itemn.startTime" class="drawer end-drawer"
                                 @mousedown="onDrawerMouseDown($event,'end',dateLane, examinrLane )"
                                 @click="onDrawerClick($event)"> {{ drawerText }}
                            </div>
                            <v-menu
                                ref="menu"
                                v-if="itemn.__bgType==='other-exam'"
                                close-on-click
                                :close-on-content-click="true"
                                class="tooltip-panel"
                            >
                                <template v-slot:activator="{ on, attrs }">
                                    <div
                                        class="other-exam-click-area"
                                        v-bind="attrs"
                                        v-on="on"
                                        @click="setActiveTooltipKey($event, itemn, 'exam')"
                                    >
                                    </div>
                                </template>
                                <exam-group-tooltip
                                    :model="activeTooltipKey === itemn && activeTooltipRole==='exam'? itemn.occupiedNonTargetExams() : null"/>
                            </v-menu>
                            <v-menu
                                ref="menu"
                                v-if="itemn.__candidate"
                                close-on-click
                                :close-on-content-click="false"
                                class="tooltip-panel"
                            >
                                <template v-slot:activator="{ on, attrs }">
                                    <div
                                        class="circleStyl"
                                        v-bind="attrs"
                                        v-on="on"
                                        @dragstart="itemn.__totalTargetRegisterCount > 0 ? onDragStart($event,itemn.firstTargetRegister() , 'candidate') : ''"
                                        @drag="itemn.__totalTargetRegisterCount > 0 ? onDrag($event) : ''"
                                        @dragend="itemn.__totalTargetRegisterCount > 0 ? onDragEnd($event) : ''"
                                        :draggable="itemn.__totalTargetRegisterCount > 0 ? true : false"
                                        v-bind:class="{
                                                    'normal-bgCndt': itemn.__candidate.__color === 'normal',
                                                    'warning-bgCndt': itemn.__candidate.__color === 'warning',
                                                    'otherExam-bgCndt': itemn.__candidate.__color === 'other-exam',
                                                    'outlineHighlightCanddt': itemn.__candidate.__searchHighlight ,
                                                }"
                                        @click="setActiveTooltipKey($event, itemn, 'candidate')"
                                    >
                                    </div>
                                </template>
                                <CandidateTooltip
                                    :model="activeTooltipKey === itemn && activeTooltipRole==='candidate'? itemn.__candidate.tooltipModel() : null"/>
                            </v-menu>
                        </div>
                    </td>
                </tr>
                <tr class="examiner-space-row">
                    <td class="c-lane-title"></td>
                    <td class="c-time-cell" v-for="(itemn, n) in modelData.timeLines"
                        @mouseup="onMouseUp($event)" :data-time-index="n"></td>
                </tr>
                </tbody>
                <tbody v-for="(OCTexm, e) in dateLane.examLanes" :key="'Y' + e" style="display: contents">
                <tr class="OCTexam-row">
                    <td class="c-lane-title">
                        <div>
                            <v-menu
                                close-on-click
                                :close-on-content-click="true"
                                offset-x
                                class="tooltip-panel"
                            >
                                <template v-slot:activator="{ on, attrs }">
                                    <div
                                        v-bind="attrs"
                                        v-on="on"
                                        @click="setActiveTooltipKey($event, OCTexm)"
                                    >
                                        {{ OCTexm.title }}
                                    </div>
                                </template>
                                <exam-group-tooltip :model="activeTooltipKey === OCTexm ? OCTexm.group.exams : null"/>
                            </v-menu>
                        </div>
                    </td>
                    <td class="c-time-cell" v-for="(itemn, n) in modelData.timeLines"
                        @mouseup="onMouseUp($event)" :data-time-index="n">
                        <div class="octBlack"
                             v-if="n != modelData.timeLines.length-1 && n >= OCTexm.start && n < OCTexm.end"></div>
                        <div class="octGray" v-else-if="n != modelData.timeLines.length-1"></div>
                    </td>
                </tr>
                <tr class="exam-space-row">
                    <td class="c-lane-title"></td>
                    <td class="c-time-cell" v-for="(itemn, n) in modelData.timeLines"
                        @mouseup="onMouseUp($event)" :data-time-index="n"></td>
                </tr>
                </tbody>
                </tbody>
                <tbody>
                <tr class="first-space-row non-top-first-space-row">
                    <td class="c-date-title">
                        <div></div>
                    </td>
                    <td class="c-lane-title"></td>
                    <td class="c-time-cell" v-for="(itemn, n) in modelData.timeLines"
                        @mouseup="onMouseUp($event)" :data-time-index="n"></td>
                    <td class="c-warnings">
                        <div></div>
                    </td>
                </tr>
                </tbody>
            </table>
        </div>
    </div>
</template>
<script>

import {Model, TransformCommand, CandidateSpeaking} from "../services/model";
import {TimeSession} from "../services/datetimes";
// import "../services/simpleTestData";
// import "../services/performanceTestData";
import {setDefaultHost} from "../services/api.js";
import CandidateTooltip from "./candidate-tooltip";
import {Config} from "../services/config";
import {DataController} from "../services/dataController";
import event_bus from "../bus";
import RenderExaminers from "./renderExaminers";
import ExamGroupTooltip from "./examGroup-tooltip";
import {FTModel, FTConfig} from "../services/freetimes/ftmodel";
import {Range} from "../services/ranges";
import {FTDataController} from "../services/freetimes/ftdataController";
import "../services/core";
import {exists, handle, visible} from "../services/core";
import "../services/user";
import {Examiner} from "../services/entities";
import {downloadContent} from "../services/files";
import JSZip from "jszip";
import Excel_export from "./excel_export";

import {examinersTimeSheetServerSideZipFile} from "../services/reports/examinersTimeSheetServerSide";
import {ExcelSheet} from "../services/excelent";

setDefaultHost("https://speaking.irsafam.com");
// setDefaultHost("http://188.121.156.114");
// setDefaultHost("http://81.91.152.6:7980");
// setDefaultHost("http://localhost:7188");

const svg = `<svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" stroke="transparent" style="position: absolute !important; background: none !important;" fill="transparent"><circle cx="7" cy="7" r="4" stroke-width="1" fill="#1d643b"/></svg>`;

const blob = new Blob([svg], {type: 'image/svg+xml'});
const url = URL.createObjectURL(blob);
const img = new Image();
img.src = url;

const mouse_bellSleep = `<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" aria-hidden="true" role="img" width="1em" height="1em" preserveAspectRatio="xMidYMid meet" viewBox="0 0 24 24"><path d="M10 21h4a2 2 0 0 1-2 2a2 2 0 0 1-2-2m11-2v1H3v-1l2-2v-6c0-3.1 2.03-5.83 5-6.71V4a2 2 0 0 1 2-2a2 2 0 0 1 2 2v.29c2.97.88 5 3.61 5 6.71v6l2 2M15 9H9v2h3.24L9 13.7V16h6v-2h-3.24L15 11.3V9z" fill="currentColor"/></svg>`
const blob1 = new Blob([svg], {type: 'image/svg+xml'});
const url1 = URL.createObjectURL(blob1);
const img_Mouse_bellSleep = new Image();
img_Mouse_bellSleep.src = url1;

const mouse_mapMarkerPlus = `<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" aria-hidden="true" role="img" width="1em" height="1em" preserveAspectRatio="xMidYMid meet" viewBox="0 0 24 24"><path d="M9 11.5A2.5 2.5 0 0 0 11.5 9A2.5 2.5 0 0 0 9 6.5A2.5 2.5 0 0 0 6.5 9A2.5 2.5 0 0 0 9 11.5M9 2c3.86 0 7 3.13 7 7c0 5.25-7 13-7 13S2 14.25 2 9a7 7 0 0 1 7-7m6 15h3v-3h2v3h3v2h-3v3h-2v-3h-3v-2z" fill="currentColor"/></svg>`
const blob2 = new Blob([svg], {type: 'image/svg+xml'});
const url2 = URL.createObjectURL(blob2);
const img_Mouse_MapMarkerPlus = new Image();
img_Mouse_MapMarkerPlus.src = url2;

export default {
    name: "index",
    components: {Excel_export, ExamGroupTooltip, RenderExaminers, CandidateTooltip},
    props: [],
    data() {
        return {
            isTargetExamExists: false,
            hoveredTdStartIndex: null,
            hoveredTdEndIndex: null,
            mDown: false,
            isDrawing: false,
            userClickedOnDrawer: null,
            mini: null,
            drawingItem: null,
            drawerText: 'f',
            currentDateLane: null,
            currentExamLane: null,
            modelData: null,
            activeTooltipKey: null,
            activeTooltipRole: null,
            isDragging: false,
            selectedRobot: '',
            dragOffsetX: 4,
            dragOffsetY: 4,
            draggingItem: null,
            mouseMoveTr: null,
            searchTimer: null,
            showExaminersOnSidebar: false,
            showExamsOnSidebar: true,
            mouseState: "default",
            searchInpTxt: null,
            miniMsg_dialog: false,
            examiner_dialog: false,
            eftModel: null,
            timeSheet_dialog: false,
            timeSheetStartDate: null,
            timeSheetEndDate: null,
            timeSheetMessage: null,
            timeSheetPreviousFileName: null,
            timeSheetPreviousFileContent: null,
            timeSheetUnderConstruction: false,
            timeSheetProgressDescTitle: null,
            timeSheetProgressDescScale: null,
            timeSheetProgressPercentage: null,
        }
    },
    computed: {
        eftDirectStart: {
            get() {
                return this.eftModel ? this.eftModel.selectedDateFreeTimesElement.__start : "";
            },
            set(val) {
                if (this.eftModel) {
                    this.eftModel.selectedDateFreeTimesElement.__start = val;
                }
            },
        },
        eftDirectEnd: {
            get() {
                return this.eftModel ? this.eftModel.selectedDateFreeTimesElement.__end : "";
            },
            set(val) {
                if (this.eftModel) {
                    this.eftModel.selectedDateFreeTimesElement.__end = val;
                }
            },
        },
        eftDirectStart2: {
            get() {
                return this.eftModel ? this.eftModel.selectedDateFreeTimesElement.__start2 : "";
            },
            set(val) {
                if (this.eftModel) {
                    this.eftModel.selectedDateFreeTimesElement.__start2 = val;
                }
            },
        },
        eftDirectEnd2: {
            get() {
                return this.eftModel ? this.eftModel.selectedDateFreeTimesElement.__end2 : "";
            },
            set(val) {
                if (this.eftModel) {
                    this.eftModel.selectedDateFreeTimesElement.__end2 = val;
                }
            },
        },
        eftNote: {
            get() {
                return this.eftModel ? this.eftModel.selectedDateFreeTimesElement.__note : "";
            },
            set(val) {
                if (this.eftModel) {
                    this.eftModel.selectedDateFreeTimesElement.__note = val;
                }
            },
        },
        eftAdminNote: {
            get() {
                return this.eftModel ? this.eftModel.selectedDateFreeTimesElement.__adminNote : "";
            },
            set(val) {
                if (this.eftModel) {
                    this.eftModel.selectedDateFreeTimesElement.__adminNote = val;
                }
            },
        },
        eftAllowVCS: {
            get() {
                return this.eftGetConstraint('vcs');
            },
            set(val) {
                this.eftSetConstraint('vcs', val);
            },
        },
        // eftAllowPaperBaseUKVI: {
        //     get() {
        //         return this.eftGetConstraint('paperBaseUKVI');
        //     },
        //     set(val) {
        //         this.eftSetConstraint('paperBaseUKVI', val);
        //     },
        // },
        eftAllowResidenceFTF: {
            get() {
                return this.eftGetConstraint('residenceFTF');
            },
            set(val) {
                this.eftSetConstraint('residenceFTF', val);
            },
        },
        eftAllowOtherCitiesFTF: {
            get() {
                return this.eftGetConstraint('otherCitiesFTF');
            },
            set(val) {
                this.eftSetConstraint('otherCitiesFTF', val);
            },
        },
    },
    methods: {
        async eft_dialog_ok() {
            try {
                await this.eftModel.submitElement.submit();
                this.examiner_dialog = false;
                this.eftModel = null;
            } catch (ex) {
                handle(ex);
            }
        },
        _stopPropagation(e) {
            e.stopPropagation();
        },
        onEftSetFull() {
            try {
                this.eftModel.selectedDateFreeTimesElement.setFull();
            } catch (ex) {
                handle(ex);
            }
        },
        onEftSetOff() {
            try {
                this.eftModel.selectedDateFreeTimesElement.setOff();
            } catch (ex) {
                handle(ex);
            }

        },
        onEftSetUnknown() {
            try {
                this.eftModel.selectedDateFreeTimesElement.setUnknown();
            } catch (ex) {
                handle(ex);
            }
        },
        eft_dialog_cancel() {
            try {
                this.examiner_dialog = false;
                this.eftModel = null;
            } catch (ex) {
                handle(ex);
            }
        },
        onExaminerTitleClick(examiner, date) {
            try {
                this.eftModel = new FTModel({
                    examiner: examiner,
                    config: new FTConfig({
                        generalTime: new Range(this.modelData.generalStartTime, this.modelData.generalEndTime),
                        fullTime: new Range(this.modelData.fullStartTime, this.modelData.fullEndTime),
                        amTime: new Range(this.modelData.amStartTime, this.modelData.amEndTime),
                        pmTime: new Range(this.modelData.pmStartTime, this.modelData.pmEndTime),
                    }),
                    initFromDate: date.nextDay(-15),
                    initToDate: date.nextDay(+16),
                });
                const index = this.eftModel._dateIndex(date);
                this.eftModel.selectDateElementAt(index);
                FTDataController.control(this.eftModel);
                this.examiner_dialog = true;
            } catch (ex) {
                handle(ex);
            }
        },
        autoPlan() {
            try {
                console.log("auto plan ...");
            } catch (ex) {
                handle(ex);
            }
        },
        onDrawerClick(e) {
            try {
                if (this.drawerText === 'v') {
                    this.drawerText = "f";
                } else {
                    this.drawerText = "v";
                }
            } catch (ex) {
                handle(ex);
            }
        },
        onTableContainerMouseOver(e) {
            try {
                if (!this.mDown) {
                    const bodyDrawers = this.$refs.tableContainer.querySelectorAll('.showDrawers');
                    for (let i = 0; i < bodyDrawers.length; i++) {
                        bodyDrawers[i].classList.remove('showDrawers');
                    }
                    const td = this.getTd(e.target);
                    if (td == null) return;
                    const tr = td.parentElement;
                    tr.classList.add("showDrawers");
                    this.drawerText = "f";
                } else {
                    this.isDrawing = true;
                }

                this.hoveredTdStartIndex = 0;
                this.hoveredTdEndIndex = 0;

                let td = this.getTd(e.target);

                if (td) {
                    if (td.dataset.spanStart == undefined || td.dataset.spanEnd == undefined) {
                        const ti = this.getTdIndex(td);
                        if (ti != null) {
                            this.hoveredTdStartIndex = ti;
                            this.hoveredTdEndIndex = ti + 1;
                        }
                    } else {
                        this.hoveredTdStartIndex = parseInt(td.dataset.spanStart);
                        this.hoveredTdEndIndex = parseInt(td.dataset.spanEnd);
                    }
                }
            } catch (ex) {
                handle(ex);
            }
        },
        getTdIndex(td) {
            if (td.dataset.timeIndex === null || td.dataset.timeIndex === undefined) return null;
            const index = parseInt(td.dataset.timeIndex);
            return index;
        },
        getTd(_target) {
            while (_target !== null && !(_target instanceof HTMLTableCellElement)) {
                _target = _target.parentNode;
            }
            return _target;
        },
        onTableContainerMouseLeave() {
            try {
                this.hoveredTdStartIndex = null;
                this.hoveredTdEndIndex = null;
                if (this.$refs.rangeTd.classList) {
                    this.$refs.rangeTd.classList.remove('show-range');
                }
                const bodyDrawers = this.$refs.tableContainer.querySelectorAll('.showDrawers');
                for (let i = 0; i < bodyDrawers.length; i++) {
                    bodyDrawers[i].classList.remove('showDrawers');
                }
            } catch (ex) {
                handle(ex);
            }
        },
        getDate(dObj) {
            return dObj.dateFormat("mmm dd");
        },
        calculateWarningClspn(dateLane) {
            let examLen = dateLane.examLanes.length * 2; //exam + space
            let examinerLen = dateLane.__examinerLaneCount * 2; //examiner + space
            let unassignedLen = 1; //unassigned + space
            return examLen + examinerLen + unassignedLen;
        },
        onDrawerMouseDown(e, pos, dateLane, examLane) {
            try {
                e.preventDefault();
                this.currentDateLane = dateLane;
                this.currentExamLane = examLane;
                this.drawingItem = pos;
                this.mDown = true;
                this.mouseMoveTr = e.target.parentElement.parentElement;
            } catch (ex) {
                handle(ex);
            }
        },
        tableMouseMove(e) {
            if (!this.mDown || !this.isDrawing) return;
            if (this.mouseMoveTr) {
                this.mouseMoveTr.classList.add("showDrawers")
            }
            const headers = this.$refs.table.tHead.rows[0].cells;
            const th = find(
                headers,
                h => {
                    const cr = h.getBoundingClientRect();

                    return cr.left <= e.clientX && e.clientX < cr.right;
                });
            if (th == null) return;
            const index = th.cellIndex - 2;
            if (index < 0 || index > this.modelData.timeCount) return;
            const time = this.modelData.realTime(index);
            if (this.drawingItem === "start") {
                this.modelData.interimTransform([
                    TransformCommand.manipulateExaminerOccupying(
                        this.currentExamLane.examiner,
                        this.modelData.__targetExam,
                        this.currentDateLane.date,
                        time,
                        this.modelData.realTime(this.currentExamLane.range.__end),
                        this.drawerText === "v" ? "vcs" : "fcs",
                        true
                    )
                ]);
            } else if (this.drawingItem === "end") {
                this.modelData.interimTransform([
                    TransformCommand.manipulateExaminerOccupying(
                        this.currentExamLane.examiner,
                        this.modelData.__targetExam,
                        this.currentDateLane.date,
                        this.modelData.realTime(this.currentExamLane.range.__start),
                        time,
                        this.drawerText === "v" ? "vcs" : "fcs",
                        true
                    )
                ]);
            }

            function find(items, condition) {
                for (const item of items)
                    if (condition(item)) return item;
                return null;
            }
        },
        onMouseUp(e) {
            try {
                this.modelData.fix();
                if (this.mDown && !this.isDrawing) {
                    this.userClickedOnDrawer = true;

                } else if (this.mDown && this.isDrawing) {
                    this.userClickedOnDrawer = false;
                }

                if (this.mouseMoveTr) {
                    this.mouseMoveTr.classList.remove("showDrawers");
                }

                this.mDown = false;
                this.isDrawing = false;
                this.currentDateLane = null;
                this.currentExamLane = null;
                this.mouseMoveTr = null;
            } catch (ex) {
                handle(ex);
            }
        },
        onDefaultMouseClick() {
            try {
                this.mouseState = 'default';
            } catch (ex) {
                handle(ex);
            }
        },
        onRegionMouseClick(e) {
            try {
                this.mouseState = 'region';
            } catch (ex) {
                handle(ex);
            }
        },
        onBreakMouseClick(e) {
            try {
                this.mouseState = 'break';
            } catch (ex) {
                handle(ex);
            }
        },
        onSideBarTargetExamClick() {
            try {
                this.mini = false;
            } catch (ex) {
                handle(ex);
            }
        },
        onExamSelectionClick(selectedExam) {
            try {
                this.modelData.transform([TransformCommand.setTargetExam(selectedExam)]);

                if (this.modelData.__targetExam) {
                    this.isTargetExamExists = true;

                } else {
                    this.isTargetExamExists = false;

                }
            } catch (ex) {
                handle(ex);
            }
        },
        setActiveTooltipKey(e, key, role) {
            try {
                this.activeTooltipKey = key;
                this.activeTooltipRole = role;
                e.stopPropagation();
            } catch (ex) {
                handle(ex);
            }
        },
        toggleDateLaneExpectedSpeakingFTF(dateLane) {
            this.modelData.transform([
                TransformCommand.setExamDateExpectedSpeakingType(
                    false,
                    this.modelData.__targetExam,
                    dateLane.date,
                    !dateLane.__expectedFTF,
                    null
                )
            ]);
        },
        toggleDateLaneExpectedSpeakingVCS(dateLane) {
            this.modelData.transform([
                TransformCommand.setExamDateExpectedSpeakingType(
                    false,
                    this.modelData.__targetExam,
                    dateLane.date,
                    null,
                    !dateLane.__expectedVCS
                )
            ]);
        },
        onDragStart(e, item, str) {
            try {
                this.draggingItem = item;
                this.draggingItem.type = str;
                if (str !== 'candidate') {
                    e.dataTransfer.setDragImage(img, 10, 10);
                }
                e.dataTransfer.setData("text", e.target.id);
            } catch (ex) {
                handle(ex);
            }
        },
        onDragEnd(e) {
            try {
                this.isDragging = false;
            } catch (ex) {
                handle(ex);
            }
        },
        onDrag(e) {
            try {
                this.isDragging = true;
            } catch (ex) {
                handle(ex);
            }
        },
        onDragOver(e) {
            try {
                e.preventDefault();
                e.dataTransfer.dropEffect = "move"
            } catch (ex) {
                handle(ex);
            }
        },
        onDrop(e, lane, item, rowStr) {
            try {
                e.preventDefault();
                if (this.draggingItem) {
                    let _examiner;
                    let _date;
                    let td = this.getTd(e.target);

                    if (td) {
                        _examiner = lane.examiner;
                        _date = lane.dateLane.date;

                        this.modelData.transform([
                            TransformCommand.setCandidateRegisterSpeaking(
                                this.modelData.__targetExam,
                                this.draggingItem.candidate,
                                new CandidateSpeaking(
                                    _date,
                                    this.modelData.realTime(parseInt(td.dataset.spanStart)),
                                    _examiner
                                )
                            )
                        ]);

                        this.draggingItem = null;
                    }
                    // this.showVmenu = false;
                }
            } catch (ex) {
                handle(ex);
            }
        },
        onExaminerRangeClick(e, examinrLane, itemn) {
            try {
                const td = this.getTd(e.target);
                if (!td) return;
                const ti = this.getTdIndex(td);
                if (e.ctrlKey) {
                    if (itemn.__bgType === 'break') {
                        this.modelData.transform([
                            TransformCommand.changeTargetExamTimeCellProperties(
                                examinrLane.examiner,
                                examinrLane.dateLane.date,
                                ti,
                                "auto-speaking",
                                null
                            )
                        ]);
                    } else {
                        this.modelData.transform([
                            TransformCommand.changeTargetExamTimeCellProperties(
                                examinrLane.examiner,
                                examinrLane.dateLane.date,
                                ti,
                                "break",
                                null
                            )
                        ]);
                    }
                } else {
                    if (this.mouseState === 'region') {
                        this.modelData.transform([
                            TransformCommand.changeTargetExamTimeCellProperties(
                                examinrLane.examiner,
                                examinrLane.dateLane.date,
                                ti,
                                null,
                                !itemn.__ispublic
                            )
                        ]);
                    } else if (this.mouseState === 'break') {
                        if (itemn.__bgType === 'break') {
                            this.modelData.transform([
                                TransformCommand.changeTargetExamTimeCellProperties(
                                    examinrLane.examiner,
                                    examinrLane.dateLane.date,
                                    ti,
                                    "auto-speaking",
                                    null
                                )
                            ]);
                        } else {
                            this.modelData.transform([
                                TransformCommand.changeTargetExamTimeCellProperties(
                                    examinrLane.examiner,
                                    examinrLane.dateLane.date,
                                    ti,
                                    "break",
                                    null
                                )
                            ]);
                        }
                    }
                }
            } catch (ex) {
                handle(ex);
            }
        },
        onSaveClick() {
            try {
                this.modelData.save();
            } catch (ex) {
                handle(ex);
            }
        },
        onSearchCndtInput(e) {
            const txt = e.target.value;

            if (this.searchTimer) {
                clearTimeout(this.searchTimer);
            }

            this.searchTimer = setTimeout(() => {
                this.modelData.transform([
                    TransformCommand.setCandidateSearchFilters({
                        text: txt,
                        useText: null,
                        regionality: null,
                        note: null,
                        warning: null
                    })
                ]);
                this.searchTimer = null;
            }, 1000);
        },
        onMiniSearchInpFocus() {
            try {
                this.mini = false;
            } catch (ex) {
                handle(ex);
            }
        },
        onEditStateClick(e) {
            try {
                if (e.ctrlKey) {
                    this.modelData.transform([
                        TransformCommand.setCandidateSearchFilters({edit: "toggle"})
                    ]);
                } else {
                    this.modelData.transform([
                        TransformCommand.setCandidateSearchFilters({edit: "alone"})
                    ]);
                }
            } catch (ex) {
                handle(ex);
            }
        },
        onUseTextStateClick(e) {
            try {
                if (e.ctrlKey) {
                    this.modelData.transform([
                        TransformCommand.setCandidateSearchFilters({useText: "toggle"})
                    ]);
                } else {
                    this.modelData.transform([
                        TransformCommand.setCandidateSearchFilters({useText: "alone"})
                    ]);
                }
            } catch (ex) {
                handle(ex);
            }
        },
        onRegionalityStateClick(e) {
            try {
                if (e.ctrlKey) {
                    this.modelData.transform([
                        TransformCommand.setCandidateSearchFilters({regionality: "toggle"})
                    ]);
                } else {
                    this.modelData.transform([
                        TransformCommand.setCandidateSearchFilters({regionality: "alone"})
                    ]);
                }
            } catch (ex) {
                handle(ex);
            }
        },
        onNoteStateClick(e) {
            try {
                if (e.ctrlKey) {
                    this.modelData.transform([
                        TransformCommand.setCandidateSearchFilters({note: "toggle"})
                    ]);
                } else {
                    this.modelData.transform([
                        TransformCommand.setCandidateSearchFilters({note: "alone"})
                    ]);
                }
            } catch (ex) {
                handle(ex);
            }
        },
        onVisibilityForceClick() {
            try {
                this.modelData.transform([
                    TransformCommand.removeAllExaminerVisibilityForces()
                ]);
            } catch (ex) {
                handle(ex);
            }
        },
        onToggleShowExamsClick() {
            try {
                if (this.showExamsOnSidebar) {
                    this.showExamsOnSidebar = false;
                    this.showExaminersOnSidebar = true;
                } else {
                    this.showExamsOnSidebar = true;
                    this.showExaminersOnSidebar = false;
                }
            } catch (ex) {
                handle(ex);
            }
        },
        resetTimeSheetPopup() {
            this.timeSheetStartDate = null;
            this.timeSheetEndDate = null;
            this.timeSheetMessage = null;
            this.timeSheetPreviousFileName = null;
            this.timeSheetPreviousFileContent = null;
            this.timeSheet_dialog = false;
        },
        onTimeSheetCancel() {
            this.resetTimeSheetPopup();
        },
        async onTimeSheetOk() {
            this.timeSheetUnderConstruction = true;
            try {
                const fromDate = Date.parseDate(this.timeSheetStartDate);
                const toDate = Date.parseDate(this.timeSheetEndDate);
                const zipFile = await examinersTimeSheetServerSideZipFile({
                    fromDate,
                    toDate,
                    inputExcelSheet: this.timeSheetPreviousFileContent,
                    progressListener: (pd, e) => {
                        const isFinished = pd.index === pd.total;
                        this.timeSheetProgressDescTitle = isFinished ? null : exists(e) ? e.title : "Admin";
                        this.timeSheetProgressDescScale = isFinished ? null : `${pd.index}/${pd.total}`;
                        this.timeSheetProgressPercentage = isFinished ? null : pd.index / pd.total;
                        if (!this.timeSheet_dialog) return false;
                    },
                });
                if (exists(zipFile)) {
                    const fileName = `${fromDate.dateFormat("yyyy/mm/dd")} - ${toDate.dateFormat("yyyy/mm/dd")}.zip`;
                    downloadContent(fileName, zipFile, "application/zip");
                }
                this.resetTimeSheetPopup();
            } catch (ex) {
                this.timeSheetMessage = ex.message;
                console.error(ex);
            } finally {
                this.timeSheetUnderConstruction = false;
            }
        },
        onTimeSheetClick() {
            this.timeSheet_dialog = true;
        },
        selectTimeSheetMonth(monthOffset) {
            const startDate = Date.today.nextMonthStart(monthOffset);
            const endDate = startDate.nextMonthStart().nextDay(-1);
            this.timeSheetStartDate = startDate.dateFormat("yyyy/mm/dd");
            this.timeSheetEndDate = endDate.dateFormat("yyyy/mm/dd");
        },
        timeSheetPreviousFileDragOver(e) {
            if (e.dataTransfer.types.includes("Files")) {
                e.preventDefault();
                e.dataTransfer.dropEffect = "copy";
            }
        },
        async timeSheetPreviousFileDrop(e) {
            if (e.dataTransfer.types.includes("Files")) {
                e.preventDefault();
                if (e.dataTransfer.files.length > 1) throw new Error("You can only select one file.");
                const file = e.dataTransfer.files[0];
                if (!file.name.endsWith(".xlsx")) throw new Error(`@InvalidFileFormat:${file.name}`);
                this.timeSheetPreviousFileName = file.name;
                this.timeSheetPreviousFileContent = await ExcelSheet.readFromBlobFirstSheet(file);
            }
        },
        timeSheetDownloadCanBeStarted() {
            return !this.timeSheetUnderConstruction && visible(this.timeSheetStartDate) && visible(this.timeSheetEndDate);
        },
        onToggleShowExaminersClick() {
            try {
                if (this.showExaminersOnSidebar) {
                    this.showExaminersOnSidebar = false;
                    this.showExamsOnSidebar = true;
                } else {
                    this.showExaminersOnSidebar = true;
                    this.showExamsOnSidebar = false;
                }
            } catch (ex) {
                handle(ex);
            }
        },
        onExaminerSelectionClick(e, examiner) {
            try {
                if (e.ctrlKey) {
                    this.modelData.transform([
                        TransformCommand.setExaminerVisibilityForce(examiner, "toggle")
                    ]);
                } else {
                    this.modelData.transform([
                        TransformCommand.setExaminerVisibilityForce(examiner, "alone")
                    ]);
                }
            } catch (ex) {
                handle(ex);
            }
        },
        fetchTheExamsBefore() {
            try {
                this.modelData.transform([TransformCommand.setExamAccessListRange(10, null)]);
            } catch (ex) {
                handle(ex);
            }
        },
        fetchTheExamsAfter() {
            try {
                this.modelData.transform([TransformCommand.setExamAccessListRange(null, 10)]);
            } catch (ex) {
                handle(ex);
            }
        },
        eftConstraintIsDisable(constraintName) {
            if (!this.eftModel.selectedDateFreeTimesElement.__isEditable) return true;
            if (!this.eftModel.selectedDateFreeTimesElement.__expectedResources[constraintName]) return true;
            return false;
        },
        eftOptionIsNoneProposed(constraintName) {
            if (!this.eftModel.selectedDateFreeTimesElement.__expectedResources[constraintName]) return true;
            return false;
        },
        eftGetConstraint(constraintName) {
            return this.eftModel ? this.eftModel.selectedDateFreeTimesElement.__examinerConstraints[constraintName] : false;
        },
        eftSetConstraint(constraintName, val) {
            if (this.eftModel && this.eftModel.selectedDateFreeTimesElement.__expectedResources[constraintName]) {
                this.eftModel.selectedDateFreeTimesElement.__examinerConstraints = this.eftModel.selectedDateFreeTimesElement.__examinerConstraints.withConstraint(constraintName, val);
            }
        },
    },
    async created() {
        const lastExamId = localStorage.getItem("cheshmeh-last-exam-id");
        const config = await Config.readConfig(lastExamId);
        this.modelData = new Model(config);


        window.model = this.modelData;


        DataController.control(this.modelData);
        console.log('model data', this.modelData);

        if (this.modelData.__targetExam) {
            this.mini = true;
            this.isTargetExamExists = true;

        } else {
            this.mini = false;
            this.isTargetExamExists = false;

        }

        window.addEventListener('beforeunload', (e) => {
            try {
                e.preventDefault();

                if (this.modelData.__anyUnsavedChange) {
                    e.returnValue = '';
                }
            } catch (ex) {
                handle(ex);
            }
        });
        window.addEventListener('keydown', (e) => {
            try {
                if (e.key === 'Escape') {
                    this.menu = false;
                    this.mouseState = 'default';
                }
            } catch (ex) {
                handle(ex);
            }
        });
    },
    mounted() {
        window.addEventListener("mouseup", e => {
            try {
                this.mDown = false;
            } catch (ex) {
                handle(ex);
            }
        });
        window.addEventListener("click", e => {
            try {
                if (this.isTargetExamExists) {
                    this.mini = true;
                }
            } catch (ex) {
                handle(ex);
            }
        });
        window.addEventListener("mousedown", e => {
            try {
                this.activeTooltipKey = null;
                this.activeTooltipRole = null;
            } catch (ex) {
                handle(ex);
            }
        });
        event_bus.$on('onTooltipDragStart', (obj) => {
            try {
                this.onDragStart(obj.ev, obj.item, obj.str);
            } catch (ex) {
                handle(ex);
            }
        });
        event_bus.$on('onTooltipDragEnd', (obj) => {
            try {
                this.onDragEnd(obj.ev);
            } catch (ex) {
                handle(ex);
            }
        });
        event_bus.$on('setTargetExam', (exam) => {
            try {
                this.modelData.transform([TransformCommand.setTargetExam(exam)]);
            } catch (ex) {
                handle(ex);
            }
        });
        event_bus.$on('onTooltipDrag', (obj) => {
            try {
                this.onDrag(obj.ev);
            } catch (ex) {
                handle(ex);
            }
        });
        event_bus.$on('onNoteInput', (obj) => {
            try {
                let _note = obj.ev.target.value;
                let _candidateRegister = obj.currentCandidate.register;

                this.modelData.transform([TransformCommand.setRegisterNote(_candidateRegister, _note)]);
            } catch (ex) {
                handle(ex);
            }
        });
    }
}
</script>
<style scoped>
.breakCursor td.occupied-time-cell {
    cursor: url('data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" aria-hidden="true" role="img" width="30px" height="30px" preserveAspectRatio="xMidYMid meet" viewBox="0 0 24 24"><svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" aria-hidden="true" role="img" width="1em" height="1em" preserveAspectRatio="xMidYMid meet" viewBox="0 0 24 24"><path d="M13.64 21.97a.99.99 0 0 1-1.33-.47l-2.18-4.74l-2.51 2.02c-.17.14-.38.22-.62.22a1 1 0 0 1-1-1V3a1 1 0 0 1 1-1c.24 0 .47.09.64.23l.01-.01l11.49 9.64a1.001 1.001 0 0 1-.44 1.75l-3.16.62l2.2 4.73c.26.5.02 1.09-.48 1.32l-3.62 1.69z" fill="white" stroke="black"/></svg><svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" aria-hidden="true" role="img" width="1em" height="1em" x="12" y="5" preserveAspectRatio="xMidYMid meet" viewBox="0 0 24 24"><path d="M10 21h4a2 2 0 0 1-2 2a2 2 0 0 1-2-2m11-2v1H3v-1l2-2v-6c0-3.1 2.03-5.83 5-6.71V4a2 2 0 0 1 2-2a2 2 0 0 1 2 2v.29c2.97.88 5 3.61 5 6.71v6l2 2M15 9H9v2h3.24L9 13.7V16h6v-2h-3.24L15 11.3V9z" fill="currentColor"/></svg></svg>') 5 0, auto;
}

.regionCursor td.occupied-time-cell {
    cursor: url('data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" aria-hidden="true" role="img" width="30px" height="30px" preserveAspectRatio="xMidYMid meet" viewBox="0 0 24 24"><svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" aria-hidden="true" role="img" width="1em" height="1em" preserveAspectRatio="xMidYMid meet" viewBox="0 0 24 24"><path d="M13.64 21.97a.99.99 0 0 1-1.33-.47l-2.18-4.74l-2.51 2.02c-.17.14-.38.22-.62.22a1 1 0 0 1-1-1V3a1 1 0 0 1 1-1c.24 0 .47.09.64.23l.01-.01l11.49 9.64a1.001 1.001 0 0 1-.44 1.75l-3.16.62l2.2 4.73c.26.5.02 1.09-.48 1.32l-3.62 1.69z" fill="white" stroke="black"/></svg><svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" aria-hidden="true" role="img" x="12" y="5" width="1em" height="1em" preserveAspectRatio="xMidYMid meet" viewBox="0 0 24 24"><path d="M9 11.5A2.5 2.5 0 0 0 11.5 9A2.5 2.5 0 0 0 9 6.5A2.5 2.5 0 0 0 6.5 9A2.5 2.5 0 0 0 9 11.5M9 2c3.86 0 7 3.13 7 7c0 5.25-7 13-7 13S2 14.25 2 9a7 7 0 0 1 7-7m6 15h3v-3h2v3h3v2h-3v3h-2v-3h-3v-2z"/> /></svg></svg>') 5 0, auto;
}

.my-table {
    border-collapse: separate;
    border-spacing: 0;
    margin-bottom: 30px;
}

.my-table th {
    position: sticky;
    top: 0;
    height: 1px;
    z-index: 1;
    background-color: white;
    width: 15px;
    font-weight: normal;
}

.my-table th:first-child {
    border: none;
}

.my-table th:nth-child(2) {
    border: none;
    width: 180px;
}

.my-table th:nth-last-child(1) {
    border-left: none;
}

.ins {
    position: relative;
    bottom: 25px;
    right: 50%;
    background-color: white;
    padding: 0 3px;
}

.ins.h-rotate {
    transform: rotate(-90deg);
}

.out {
    height: -webkit-fill-available;
    margin-top: 45px;
    border-bottom: 5px solid transparent;
    width: inherit;
    border-left: 1px dotted lightgray;
}

.my-table td {
    height: 1px;
}

.c-date-title {
    width: 100px;
    position: relative;
}

.c-date-est {
    position: absolute;
    right: 2px;
    top: 2px;
    border: 1px solid #00000050;
    background-color: #00000090;
    padding: 1px;
    font-size: 50%;
    line-height: 1;
    display: none;
    align-items: center;
    justify-content: center;
}

.c-date-title:hover .c-date-est,
.c-date-est.c-date-est-any {
    display: flex;
}

.c-date-est > * {
    background-color: orangered;
    color: white;
    padding: 1px 3px;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
}

.c-date-est > .est-off {
    opacity: 0.4;
}

.c-date-est > *:first-child {
    margin-inline-end: 2px;
}

.c-date-title-content {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    height: 100%;
}

.c-date-title-content > *:nth-child(2) {
    font-size: smaller;
    color: yellow;
}

.c-date-title-content > *:nth-child(3) {
    font-size: 90%;
    margin-top: 3px;
    opacity: 0.3;
}

.c-date-title-content:hover > *:nth-child(3) {
    opacity: 1;
}

.c-lane-title {
    width: 180px;
    padding-left: 5px;
    padding-right: 5px;
}

.examiner-row {
    height: 14px;
}

.examiner-row .c-lane-title > div {
    display: flex;
    justify-content: flex-end;
}

.end-occupied-time-cell {
    border-right: 1px solid black;
}

.start-occupied-time-cell {
    border-left: 1px solid black;
}

.occupied-time-cell {
    border-top: 1px solid black;
    border-bottom: 1px solid black;
}

.occupied-session:not(.bgType-otherExam):not(.bgType-notFree) {
    background-color: #ffa500a0;
}

.occupied-session.bgType-otherExam {
    background-color: transparent;
    background-image: linear-gradient(135deg, #4169e1a0 50%, #ffa500a0 50%);
}

.occupied-session.bgType-notFree {
    background-color: transparent;
    background-image: linear-gradient(135deg, #00000060 50%, #ffa500a0 50%);
}

.occupied-time-cell.t-vcs > *::after {
    content: "v";
    position: absolute;
    left: 3px;
    top: 0;
    color: #000000a0;
    font-family: 'Times New Roman';
    font-style: italic;
}

.examiner-row .c-time-cell:not(.occupied-time-cell):not(.bgType-none) {
    border-top: 1px dotted #00000010;
    border-bottom: 1px dotted #00000010;
}

.c-time-cell:not(.start-occupied-time-cell) {
    border-left: 1px dotted lightgray;
}

.occupied-time-cell > *::before {
    position: absolute;
    content: "";
    width: calc(100% - 2px);
    height: 2px;
    top: -3px;
    left: 1px;
}

.examiner-row .c-time-cell > div {
    height: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
}

.bgType-none {

}

.bgType-notFree {
    background-image: repeating-linear-gradient(135deg, #00000040 0 1px, #00000060 2px 3px);
}

.bgType-break {
    /*background-image: repeating-linear-gradient(135deg, white 0 1px, #ffa500a0 2px 3px);*/
    /*background-color: #f0f0f0;*/
}

/*tr.examiner-row.showDrawers*/
/*td*/
.bgType-free {
    background-color: #f0f0f0;
}

.bgType-otherExam {
    background-image: repeating-linear-gradient(135deg, #4169e160 0 1px, #4169e1A0 2px 3px);
}

.c-time-cell > * {
    position: relative;
}

.time-cell-issue > *::before {
    background-color: red;
}

.rg-regional:not(.time-cell-issue) > *::before {
    background-color: #4169e1;
}

.rg-non-regional:not(.time-cell-issue) > *::before {
    background-color: #A1B9F1;
}

.first-space-row {
    height: 8px;
}

.non-top-first-space-row td {
    border-top: 3px solid gray;
}

.examiner-space-row {
    height: 6px;
}

.exam-space-row {
    height: 3px;
}

.headerBorderBottom th {
    border-bottom: 2px solid gray !important;
}

.headerBorderBottom th:last-child {
    border-bottom: 2px solid gray !important;
}

.examiner-color {
    width: 12px;
    height: 12px;
    display: inline-block;
    border: 1px solid #676767;
    border-radius: 3px;
    margin-right: 4px;
    margin-left: 1px;
}

.OCTexam-row .c-lane-title > div {
    color: rgba(0, 0, 0, 0.5);
    font-size: smaller;
    display: flex;
}

.underHeader-row td {
    background-color: inherit;
    border-right: 1px dotted gray;
    height: 19px;
    border-bottom: none;
    border-top: none;
}

.show-range {
    border-bottom: 5px solid black !important;
}

/*.tooltip-container {*/
/*    border-radius: 4px;*/
/*    backdrop-filter: blur(15px);*/
/*    padding: 10px;*/
/*    box-shadow: 0 0 3px 4px #ff000050;*/
/*}*/

/*.linkBtn {*/
/*    box-shadow: none;*/
/*    text-decoration: underline;*/
/*    border-radius: unset;*/
/*    background-color: inherit !important;*/
/*    text-transform: none;*/
/*    min-width: 20px !important;*/
/*    padding-right: 0 !important;*/
/*    padding-left: 0 !important;*/
/*    color: #007eff;*/
/*}*/

.v-menu__content {
    box-shadow: 0 0 3px 2px #000000A0;
}

.highlight {
    color: red !important;
}

.inactive {
    color: #383d41 !important;
}

.exam-bg .c-date-title {
    background-color: black;
    color: white;
}

.highLight-bg .c-date-title {
    background-color: #4169e1FF;
    color: white;
}

.warning-bg .c-date-title {
    background-color: red;
    color: black;
}

.warning-td {
    color: red;
    background-color: white !important;
}

.inactive-bg .c-date-title {
    background-color: gray;
    color: white;
}

.searchCandidateInp {
    border: 1px solid #e1e1e1;
    width: 100%;
    border-radius: 3px;
    outline: none;
    padding: 3px;
    float: left;
    display: inline-block
}

.searchCandidateInp::placeholder {
    color: #dfdfdf;
}

.circleStyl {
    position: absolute;
    width: 8px;
    height: 8px;
    border-radius: 9px;
    border: 1px solid;
}

.other-exam-click-area {
    position: absolute;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
}

.normal-bgCndt {
    background-color: black;
    border-color: gray;
}

.warning-bgCndt {
    background-color: red;
    border-color: black;
}

.otherExam-bgCndt {
    background-color: #9e9e9e;
    border-color: black;
}

.outlineHighlight {
    position: relative;
}

.outlineHighlight::after {
    content: "";
    position: absolute;
    width: 100%;
    height: 0px;
    top: -3px;
    left: 0;
    border-radius: 6px;
    background-color: #ff000050;
    z-index: 2;
    animation: Blinking-Bar 0.7s alternate infinite;
}

.outlineHighlightCanddt {
    z-index: 2;
    animation: Blinking-Circle 0.7s alternate infinite;
}

@keyframes Blinking-Circle {
    0% {
        box-shadow: 0 0 3px 4px #ff000050;
    }

    100% {
        box-shadow: 0 0 3px 5px #ff000050;
    }
}

@keyframes Blinking-Bar {
    0% {
        box-shadow: 0 0 3px 1px #ff000050;
    }

    100% {
        box-shadow: 0 0 3px 2px #ff000050;
    }
}

tr.showDrawers .start-drawer,
tr.showDrawers .end-drawer {
    display: flex;
}

.start-drawer,
.end-drawer {
    position: absolute;
    content: "";
    width: 15px;
    height: 300%;
    top: -160%;
    background-color: #4169e1A0;
    backdrop-filter: blur(2px);
    border-radius: 3px;
    border: 1px solid lightgray;
    box-sizing: border-box;
    z-index: 3;
    display: none;
    color: transparent;
    font-weight: bold;
    justify-content: center;
    align-items: start;
    font-size: 120%;
    left: -4px;
    padding-top: 3px;
}

.start-drawer:hover,
.end-drawer:hover {
    color: white;
}

.currentExamHighlight {
    background-color: royalblue !important;
    color: white !important;
}

.examList.new-date-exam {
    border-top: 1px dashed #CCCCCC;
    padding-top: 3px;
    margin-top: 2px;
}

.examList.mock-exam {
    opacity: 0.3;
}

.exam-list-color {
    border: 1px solid lightgray;
    border-radius: 3px;
    margin-right: 5px;
    width: 12px;
    height: 12px;
}

.exam-list-title {
    display: flex;
    align-items: center;
}

.exam-list-title-ir {
    margin-inline-start: 5px;
    font-size: 90%;
    opacity: 0.3;
}

.exam-list-title:hover > .exam-list-title-ir {
    opacity: 0.6;
}

.examList {
    padding-left: 5px;
    padding-top: 2px;
    padding-bottom: 2px;
    cursor: pointer;
    display: flex;
    align-items: center;
}

.octBlack,
.octGray {
    background-color: maroon;
    width: inherit;
    height: 3PX;
}

.octBlack {
    opacity: 0.3;
}

.octGray {
    opacity: 0.1;
}

.saveBtnContainer {
    width: 100% !important;
    padding: 5px;
}

.saveBtn {
    height: 30px;
    width: 100%;
    align-items: center;
    border-radius: 3px;
    display: inline-flex;
    flex: 0 0 auto;
    letter-spacing: .0892857143em;
    justify-content: center;
    outline: 0;
    position: relative;
    text-decoration: none;
    text-indent: 0.0892857143em;
    text-transform: uppercase;
    transition-duration: .28s;
    transition-property: box-shadow, transform, opacity;
    transition-timing-function: cubic-bezier(.4, 0, .2, 1);
    -webkit-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    user-select: none;
    vertical-align: middle;
    white-space: nowrap;
    cursor: pointer;
    background-color: #4caf50 !important;
    border-color: #4caf50 !important;
    color: #fff;
}

.saveBtn.disableSaveBtn {
    background-color: #b8ddb9 !important;
    cursor: default;
}

.scrollSideBar {
    flex-grow: 1;
    overflow-y: scroll;
    margin-left: 5px;
    margin-right: 5px;
    padding: 3px;
    border-radius: 3px;
    border: 1px dashed #E0E0E0;
    background-color: #FAFAFA;
}

.scrollSideBar::-webkit-scrollbar {
    display: none;
}

/* Hide scrollbar for IE, Edge and Firefox */
.scrollSideBar {
    -ms-overflow-style: none; /* IE and Edge */
    scrollbar-width: none; /* Firefox */
}

.showCount {
    text-decoration: underline;
    color: #e34848;
}

.showCount-0 {
    color: lightgray;
}

.showCount.search-result-long,
.showCount-0.search-result-long {
    white-space: nowrap;
    font-size: smaller;
}

.boldTxt {
    /*font-weight : 600;*/
}

.cDataTitle {
    width: 100px;
    text-align: right;
}

.examiner-title > * {
    padding-left: 3px;
    float: right;
    cursor: pointer;
}

.examinerTitle-containor {
    background-image: linear-gradient(to right, orange 0 30%, transparent);
}

.unplannedContainer {
    height: 100%;
    display: flex;
    flex-direction: column;
    justify-content: flex-end;
    padding-bottom: 3px;
    padding-right: 10px;
}

.unplannedContainer > * {
    display: flex;
    justify-content: space-between;
}

.unplannedTxt {
    color: #e34848;
}

.h-warnings {
    width: 200px;
    padding-bottom: 3px;
}

.h-warnings > div {
    height: 100%;
    display: flex;
    flex-direction: column;
    justify-content: flex-end;
}

.h-warnings > div > div {
    display: flex;
    margin-bottom: 4px;
}

.c-warnings {
    width: 200px;
    padding-top: 8px;
}

.c-warnings > div {
    height: 100%;
    display: flex;
    flex-direction: column;
    overflow-y: auto;
}

.c-warnings > div::-webkit-scrollbar {
    width: 2px;
}

.c-warnings > div::-webkit-scrollbar-thumb {
    background-color: maroon;
}

.c-warnings > div > div {
    display: flex;
    margin-bottom: 4px;
}

.warningTxt.w-detail {
    text-decoration: underline;
}

.warningTxt.wt-error {
    color: red;
}

.warningTxt.wt-warning {
    color: #ff000070;
}

.warningTxt.wt-hint {
    color: black;
}

.warningTxt.wt-inform {
    color: gray;
}

.warningTxt::before {
    content: "\25CF\00A0";
}

.hide {
    display: none !important;
}

.invisible {
    visibility: hidden;
}

.miniSide {
    display: flex;
    flex-direction: column;
    width: 50px;
    height: 100vh;
    right: 0;
    z-index: 4;
    background-color: white;
    position: fixed;
    padding-top: 5px;
    border-left: 1px dotted lightgray;
}

.miniSide > * {
    margin-bottom: 2px;
}

.maxSide {
    display: flex;
    flex-direction: column;
    width: 250px;
    height: 100vh;
    right: 0;
    z-index: 4;
    background-color: white;
    position: fixed;
    border-left: 1px dotted lightgray;
}

.symbol-containor {
    width: 20px;
    height: 20px;
    display: flex;
    justify-content: center;
    align-items: center;
}

.symbol-containor.disabledStyl {
    opacity: 0.3;
}

.selectable-button,
.command-button {
    border: 1px solid gray;
    border-radius: 3px;
    width: 20px;
    height: 20px;
    padding: 2px;
    display: flex;
    justify-content: center;
    align-items: center;
}

.selectable-button {
    background-color: #ffa500;
}

.command-button {
    background-color: royalblue;
}

.command-button > .v-icon {
    color: white;
}

.selectable-button.disabledStyl,
.comman-button.disabledStyl {
    opacity: 0.3;
    background-color: #E5E5E5;
}

.selectable-button > *,
.command-button > * {
    font-size: 14px;
}

.symbol {
    font-size: 18px;
}

.tools-folder {
    display: flex;
    padding-left: 5px;
    padding-right: 5px;
}

.tools-folder > * {
    margin-bottom: 1px;
}

.list-buttons-folder {
    margin-bottom: 3px;
}

.miniSide .tools-folder {
    flex-direction: column;
}

.maxSide .tools-folder > * {
    margin-right: 2px;
}

.mouse-pointer-folder {
    background-color: #FAFAFA;
    border-top: 1px dashed #c5c5c580;
    border-bottom: 1px dashed #c5c5c580;
}

.miniSide .mouse-pointer-folder {
    padding-top: 2px;
    padding-bottom: 2px;
}

.maxSide .mouse-pointer-folder {
    margin-top: 5px;
    margin-bottom: 5px;
    padding-top: 4px;
    padding-bottom: 4px;
}

.miniSide .factor {
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.miniSide .factor > *:nth-child(2) {
    font-size: smaller;
}

.targetExamSideBarContainer {
    justify-content: center;
    display: flex;
    flex-grow: 1;
}

.targetExamSideBar {
    -ms-writing-mode: tb-lr;
    writing-mode: vertical-lr;
    display: flex;
    justify-content: center;
    font-size: larger;
    font-weight: bolder;
}

.search-options {
    justify-content: space-between;
    margin-top: 2px;
}

.search-buttons {
    display: flex;
}

.search-buttons > * {
    margin-right: 2px;
}

.mu-10 {
    margin-top: 40px;
}

.iconTooltip {
    font-size: 100%;
    background-color: white;
    color: black;
    border: 1px dashed gray;
    padding: 4px;
    line-height: 100%;
}

.auto-plan {
    text-decoration: underline;
    color: gray;
    font-size: smaller;
}

.auto-plan > * {
    cursor: pointer;
}

.tooltip-panel {
    z-index: 5;
}

.waiting-symbol {
    border-radius: 50%;
}

.running-process .waiting-symbol {
    animation: Blinking-Circle 0.7s alternate infinite;
}

.fetch-more-button {
    cursor: pointer;
    display: flex;
    justify-content: center;
    text-decoration: underline;
    height: 15px;
    background-color: #00000010;
    opacity: 0.5;
}

.fetch-more-button:hover {
    opacity: 1;
}

.eft_container {
    padding: 15px;
    background-color: white;
}

.eft-table-container {
    overflow-y: scroll;
    max-height: 500px;
}

.eft-table-container::-webkit-scrollbar {
    display: none;
}

.eft-table-container td {
    height: 1px;
}

.msg_container {
    padding: 15px;
    width: 800px;
    background-color: white;
    height: fit-content;
}

.msg_container .msg_Title {
    display: inline-block;
}

.msg_container .msg_header {
    border-bottom: 1px solid gray;
    padding-bottom: 5px;
}

.msg_container .subMsg_Title {
    margin-left: 35px;
}

.menu_down_icon {
    transform: rotate(-90deg);
}

.menu_down_icon_open {
    transform: none;
}

.eft-ranges {
    width: 500px;
    height: 1px;
}

.eft-ranges > * {
    display: flex;
    height: 100%;
}

.eft-range {
    display: flex;
    justify-content: center;
    align-items: center;
    overflow: hidden;
    font-size: smaller;
    white-space: nowrap;
}

.eft-range-direct {
    background-color: royalblue;
    color: white;
}

.eft-range-template {
    background-color: lightblue;
}

.eft-range-unknown {
    background-image: repeating-linear-gradient(135deg, #00000040 0 1px, #00000060 2px 3px);
}

.eft-date {
    cursor: pointer;
}

.eft-date.eft-date-selected {
    background-color: gold;
}

.eft-date:not(.eft-date-selected):hover {
    background-color: #00000010;
}

.eft-date > td {
    padding-top: 3px;
    padding-bottom: 3px;
}

.eft-date > td:nth-child(1) {
    padding-right: 4px;
    padding-left: 10px;
    text-align: right;
    color: royalblue;
}

.eft-table-container td:nth-child(2) {
    padding-right: 2px;
    border-right: 2px solid black;
}

.eft-table-container td:nth-child(3) {
    border-right: 2px solid black;
}

.eft-date-status {
    width: 30px;
}

.eft-date-status > * {
    display: flex;
    justify-content: center;
    align-items: center;
}

.eft-date-status > * > * {
    font-size: 11px;
}

.eft-date-error {
    color: #ff0000A0;
}

.eft-date-warning {
    color: orange;
}

.eft-date-normal {
    visibility: hidden;
}

.eft-content {
    display: flex;
}

.eft-control-container {
    width: 150px;
    padding-left: 5px;
    margin-left: 5px;
    border-left: 1px dotted lightgray;
}

.eft-control-container > * {
    margin-bottom: 20px;
}

.eft-control-title {
    font-size: 130%;
    font-family: "Times New Roman";
    font-weight: bold;
    margin-bottom: 5px;
    display: flex;
}

.eft-control-role {
    color: orange;
}

.eft-control-values {
    display: flex;
}

.eft-control-ranges {
    display: grid;
    grid-template-columns: max-content max-content;
    grid-template-rows: max-content max-content;
    grid-gap: 5px;
}

.eft-control-ranges > input {
    width: 45px;
    border: 1px solid lightgray;
    border-radius: 3px;
    padding: 3px;
    text-align: center;
}

.eft-control-ranges > input:disabled {
    background-color: #f0f0f0;
}

.eft-control-ranges > input::placeholder {
    opacity: 0.5;
}

.eft-control-note {
    margin-top: 5px;
}

.eft-control-note:not(.eft-control-admin-note) > input {
    color: maroon;
}

.eft-control-note > input {
    width: 100%;
    border: 1px solid lightgray;
    border-radius: 3px;
    padding: 3px;
}

.eft-control-note > input:disabled {
    background-color: #f0f0f0;
}

.eft-control-note > input::placeholder {
    opacity: 0.5;
}

.eft-control-constraints {
    margin-top: 5px;
    display: flex;
    flex-direction: column;
    font-size: 80%;
}

.eft-control-constraints > * {
    display: flex;
    margin-bottom: 2px;
    padding: 2px 0;
    align-items: center;
}

.eft-control-constraints > .disable-style {
    background-color: #efefef;
}

.eft-control-constraints > .none-proposed-style {
    color: lightgray;
}

.eft-control-constraints > * > input[type=checkbox] {
    margin-right: 2px;
    width: 10px;
    height: 10px;
}

.eft-control-commands {
    padding-left: 5px;
    display: grid;
    grid-auto-flow: column;
    grid-template-rows: max-content max-content max-content;
    grid-gap: 3px;
}

.eft-control-commands > * {
    text-decoration: underline;
    cursor: pointer;
    font-size: smaller;
}

.eft-control-commands > :not(.rare-link) {
    color: blue;
}

.eft-control-commands > .rare-link {
    color: gray;
    margin-top: 3px;
}

.eft-dialog-buttons {
    display: flex;
    justify-content: flex-end;
    padding-top: 5px;
    padding-bottom: 5px;
}

.button-div {
    min-width: 60px;
    padding: 5px 10px;
    border-radius: 3px;
    cursor: pointer;
    display: flex;
    justify-content: center;
    align-items: center;
}

.eft-cancel-button {
    background-color: lightgray;
    border: 1px solid gray;
}

.eft-ok-button {
    background-color: royalblue;
    color: white;
    border: 1px solid gray;
    margin-left: 5px;
}

.eft-ok-button-disabled {
    opacity: 0.5;
}

.eft-dialog-title {
    font-size: 130%;
    font-family: "Times New Roman";
    font-weight: bold;
    margin-bottom: 15px;
    display: flex;
    justify-content: center;
}

.eft-dialog-title-examiner {
    color: darkorange;
}

.eft-dialog-title-range {
    margin-left: 15px;
}

.copyright-style {
    width: 100%;
    position: fixed;
    bottom: 0;
    padding: 5px 0;
    background-color: white;
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 1;
}

.copyright-style > * {
    color: white;
    background-color: royalblue;
    padding: 2px 5px;
    border-radius: 2px;
}

.time-sheet-dialog {
    display: flex;
    flex-direction: column;
    padding: 5px;
    font-size: 15px;
    border: 1px solid black;
}

.time-sheet-dialog-header {
    display: flex;
    align-items: center;
    background-color: royalblue;
    color: white;
    padding: 5px;
    font-size: larger;
    border-top-left-radius: 3px;
    border-top-right-radius: 3px;
}

.time-sheet-dialog-body {
    display: flex;
    flex-direction: column;
}

.time-sheet-dialog-content {
    display: flex;
    padding: 20px 30px 5px 5px;
}

.time-sheet-dialog-content-main {
    display: grid;
    grid-template-columns: max-content max-content;
    grid-template-rows: max-content max-content;
    column-gap: 5px;
    row-gap: 5px;
}

.time-sheet-previous-export-drop-area {
    padding: 5px;
    border: 1px dashed lightgray;
    border-radius: 3px;
}

.time-sheet-previous-export-drop-area-preview-text {
    display: flex;
    justify-content: center;
    font-size: 70%;
    opacity: 0.6;
}

.time-sheet-previous-export-drop-area-preview-text > * {
    display: flex;
    justify-content: center;
    align-items: center;
}

.time-sheet-factor-title {
    display: flex;
    align-items: center;
    justify-content: flex-end;
    font-weight: bold;
    font-size: smaller;
}

.time-sheet-dialog-content-main-data > input {
    border: 1px solid lightgray;
    border-radius: 3px;
    font-size: smaller;
    padding: 2px;
}

.time-sheet-dialog-content-shortcuts {
    margin-inline-start: 5px;
    display: grid;
    grid-template-columns: max-content max-content;
    grid-template-rows: max-content max-content;
    column-gap: 5px;
    row-gap: 5px;
}

.time-sheet-dialog-content-shortcuts-month {
    font-size: 70%;
    text-decoration: underline;
    color: blue;
    cursor: pointer;
}

.link-button {
    color: blue;
    text-decoration: underline;
    cursor: pointer;
}

.time-sheet-dialog-buttons {
    display: flex;
    justify-content: flex-end;
    font-size: smaller;
}

.time-sheet-dialog-buttons > *:not(:first-child) {
    margin-inline-start: 3px;
}

.time-sheet-dialog-ok-button,
.time-sheet-dialog-cancel-button {
    min-width: 60px;
    min-height: 25px;
    padding: 5px;
    border: 1px solid lightgray;
    border-radius: 3px;
}

.time-sheet-dialog-ok-button {
    background-color: green;
    color: white;
}

.time-sheet-dialog-ok-button.disable-style {
    opacity: 0.5;
}

.time-sheet-dialog-cancel-button {
    background-color: lightgray;
    color: white;
}

.time-sheet-dialog-message {
    color: red;
    font-size: smaller;
}

.time-sheet-dialog-content-progress {
    display: flex;
    flex-direction: column;
}

.time-sheet-dialog-content-progress-desc {
    display: flex;
    justify-content: space-between;
    font-size: 60%;
}

.time-sheet-dialog-content-progress-bar {
    min-height: 5px;
    margin-top: 5px;
    margin-bottom: 5px;
    border: 1px solid gray;
}

.time-sheet-dialog-content-progress-bar-done {
    background-color: lightgray;
    min-height: 5px;
}

</style>
