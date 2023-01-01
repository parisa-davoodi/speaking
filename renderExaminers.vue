<template>
    <tr class="examiner-row">
        <td class="c-lane-title">
            <div v-bind:class="{'examinerTitle-containor': examinrLane.__bgColor === 'force'}">
                {{ examinrLane.examiner.title }}
            </div>
        </td>
        <td class="c-time-cell" v-for="(itemn, n) in examinrLane.__timeCells" :key="'Z' + n"
            v-bind:class="{
                                        'start-occupied-time-cell': examinrLane.range.__start == itemn.startTime,
                                        'end-occupied-time-cell': examinrLane.range.__end == itemn.endTime,
                                        'occupied-time-cell': itemn.__type !== 'external',
                                        'bgType-none': itemn.__bgType == 'none',
                                        'bgType-free': itemn.__bgType == 'free',
                                        'bgType-notFree': itemn.__bgType == 'not-free',
                                        'bgType-otherExam': itemn.__bgType == 'other-exam',
                                        'bgType-break': itemn.__bgType == 'break',
                                        'bgType-thisExam': itemn.__bgType == 'this-exam',
                                        'bgType-isNotPublic': itemn.__ispublic == false,
                                     }"
            @mouseup="onMouseUp($event)"
            @drop="onDrop($event,examinrLane,itemn)"
            @dragover="onDragOver($event)"
            @click="examinrLane.range.__start <= n && n < examinrLane.range.__end ? onExaminerRangeClick($event,examinrLane,itemn) : ''"
            :colspan="itemn.timeCount"
            :data-span-start="itemn.startTime"
            :data-span-end="itemn.endTime"
        >
            <div class="c-time-cell-div">
                <div v-if="examinrLane.range.__start === itemn.startTime" class="drawer"
                     @mousedown="onDrawerMouseDown($event,'start',dateLane, examinrLane )"
                     @click="onDrawerClick($event)"> {{ drawerText }}
                </div>
                <div v-if="examinrLane.range.__end === itemn.startTime" class="drawer"
                     @mousedown="onDrawerMouseDown($event,'end',dateLane, examinrLane )"
                     @click="onDrawerClick($event)">
                </div>
                <v-menu
                    :value="showVmenu"
                    v-if="itemn.__candidate"
                    close-on-click
                    :close-on-content-click="false"
                    style="z-index: 30000"
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
                                                }">
                        </div>
                    </template>
                    <CandidateTooltip :model="itemn.__candidate.tooltipModel()"/>
                </v-menu>
            </div>
        </td>
    </tr>
</template>

<script>
export default {
    name: "renderExaminers",
    props: ['examinrLane'],
    data() {
        return {
        }
    },

}
</script>

<style scoped>

</style>
