<!--{{{ Pug -->
<template lang="pug">

div.FretboardSettings

	//----------------------------------------------------------------------
	//- Instrument settings
	//----------------------------------------------------------------------
	component(
		:is="isMobileDevice ? 'div' : 'VPopupMenu'"

		icon="guitar"
		text="Instrument"
		)
		div.settings-menu

			//- Instrument
			VSelect(
				:options="instruments"
				icon="guitar"
				is-contained
				is-first-item

				v-model="instrument"
				)

			div.settings-menu__separator

			//- Tuning
			VSelect(
				:options="tunings"
				icon="music"
				is-contained

				v-model="tuning"
				)

			div.settings-menu__separator

			//- Capo
			VSelect(
				:options="capoFrets"
				icon="circle-arrow-up"
				is-contained
				is-last-item

				v-model.number="capo"
				)

	//----------------------------------------------------------------------
	//- Display settings
	//----------------------------------------------------------------------
	component(
		:is="isMobileDevice ? 'div' : 'VPopupMenu'"

		icon="eye"
		text="Display"
		)
		div.settings-menu

			//- Theme
			VSelect(
				:options="{ light: 'Light mode', dark: 'Dark mode', system: `System colors (${isSystemDarkModeOn ? 'dark' : 'light'})` }"
				:icon="darkModeSetting == 'system' ? 'desktop-alt' : isDarkModeOn ? 'moon' : 'sun'"
				is-contained

				v-model="darkModeSetting"
				)

			div.settings-menu__separator

			//- Fretting hand
			VToggleButton(
				v-if="false"
				text="Right-handed fretting"
				icon="hand"
				:is-active="isFlippedHor"

				@click.native="$store.commit('fretboard/toggle.isFlippedHor')"
				)

			//- Fretting hand
			VToggleButton(
				v-if="false"
				text="Mirror vertically"
				icon="arrows-up-down"
				:is-active="isFlippedVert"

				@click.native="$store.commit('fretboard/toggle.isFlippedVert')"
				)

			div.settings-menu__separator

			//- Fret numbers
			VToggleButton(
				text="Show fret numbers"
				icon="list-ol"
				:is-active="isShowingFretNbs"

				@click.native="$store.commit('fretboard/toggle.isShowingFretNbs')"
				)

			//- Infos displayed on the notes:
			//-   * single sequence:    nothing/note name/interval (select menu)
			//-   * multiple sequences: nothing/note name          (toggle button)
			VSelect(
				v-if="displayedSequences.length <= 1"

				:options="{ none: 'nothing', name: 'names', degree: 'degrees', interval: 'intervals' }"
				icon="circle-info"
				label-prefix="Display on notes:"
				is-contained

				v-model="noteInfos"
				)
			VToggleButton(
				v-else

				text="Show note names"
				icon="circle-info"
				:is-active="isShowingNoteNames"

				@click.native="$store.commit('fretboard/toggle.isShowingNoteNames')"
				)

			div.settings-menu__separator

			//- Fret range
			div.fret-range(v-if="false")
				VMultiRange.fret-range__slider(
					:min="0"
					:max="MAX_NB_FRETS - 1"
					:min-gap="MIN_NB_FRETS"
					:is-flipped="isFlippedHor"

					:values="fretRangeDisplay"

					@input=" fretRangeDisplay = $event"
					@change="fretRange = fretRangeDisplay = $event"
					)
				p.fret-range__text
					span(v-html="lowestFret")
					span.fret-range__text__separator ⟷
					span(v-html="highestFret")

	//----------------------------------------------------------------------
	//- Export menu
	//----------------------------------------------------------------------
	VPopupMenu(
		v-if="!isMobileDevice"

		icon="file-arrow-down"
		text="Export"

		:force-closing="exportMenuClose"
		)
		div.export-menu
			div.export-menu__text
				p: strong Choose a format to export in
				p: em
					| If you want to print this  fretboard, choose PNG or JPG.
					| The SVG format is most useful for embedding in web pages as it will scale perfectly when resized.
			div.export-menu__buttons
				each format of ['png', 'jpg', 'svg']
					VButton(
						icon=(format == 'svg' ? 'file-image' : 'camera')
						text=format.toUpperCase()

						@click=`exportFretboard('${format}')`
						)

</template>
<!--}}}-->


<!--{{{ JavaScript -->
<script>

import { get, sync }                          from 'vuex-pathify'

import { MIN_NB_FRETS, MAX_NB_FRETS }         from '@/modules/constants'
import { mapObjectToObject }                  from '@/modules/object'
import { instruments, tunings, tuningsNames } from '@/modules/music'
import { getFrets }                           from '@/modules/fretboard'
import { formatOrdinalSuffix, formatFretNb }  from '@/modules/text'

export default {
	name: 'FretboardSettings',

	data() {
		return {
			exportMenuClose:  false,
			fretRangeDisplay: this.$store.state.fretboard.fretRange,
		}
	},

	computed: {
		lowestFret() {
			return formatOrdinalSuffix(formatFretNb(this.fretRangeDisplay[this.isFlippedHor ? 1 : 0]));
		},
		highestFret() {
			return formatOrdinalSuffix(formatFretNb(this.fretRangeDisplay[this.isFlippedHor ? 0 : 1]));
		},
		tunings() {
			return mapObjectToObject(tunings[this.instrument], tuning => tuningsNames[tuning]);
		},
		...sync([
			'darkModeSetting',
		]),
		...sync('fretboard', [
			'instrument',
			'tuning',
			'capo',
			'fretRange',

			'noteInfos',

			'isShowingFretNbs',
			'isShowingNoteNames',
			'isFlippedHor',
			'isFlippedVert',
		]),
		...get([
			'sequences/displayedSequences',

			'isDarkModeOn',
			'isMobileDevice',
			'isSystemDarkModeOn',
		]),
	},

	created() {
		this.MIN_NB_FRETS = MIN_NB_FRETS;
		this.MAX_NB_FRETS = MAX_NB_FRETS;

		this.instruments  = instruments;
		this.capoFrets    = Object.fromEntries([[0, 'No capo'], ...[...Array(11).keys()].map(fret => [fret + 1, `Capo ${formatFretNb(fret + 1)}`])]);
	},

	methods: {
		async exportFretboard(format) {
			this.exportMenuClose = !this.exportMenuClose;

			const { exportFretboard } = await import('../modules/export-img.js');
			exportFretboard(
				format,
				this.$store.state.sequences.sequences,
				getFrets(this.displayedSequences, (a => { if (!this.isFlippedVert) a.reverse(); return a; })([...tunings[this.instrument][this.tuning]]), this.capo),
				instruments[this.instrument].nbStrings,
				this.fretRange[0],
				this.fretRange[1],
				(this.displayedSequences.length > 1) ? (this.isShowingNoteNames ? 'name' : 'none') : this.noteInfos,
				this.isFlippedHor,
				this.isShowingFretNbs,
				this.darkModeSetting == 'system' ? this.isSystemDarkModeOn : this.isDarkModeOn,
				format != 'svg',
			);
		},
	}
}

</script>
<!--}}}-->


<!--{{{ SCSS -->
<style lang="scss" scoped>

.FretboardSettings {
	@include mq($until: desktop)
	{
		display: flex;
		align-items: stretch;
		flex-direction: column;
	}

	@include mq($from: desktop)
	{
		display: flex;
		@include space-children-h(10px);
	}
}

.settings-menu {
	display: flex;
	flex-direction: column;
	justify-content: stretch;

	min-width: 250px;
}

.settings-menu__separator {
	display: none;

	height: 1px;

	background-color: var(--color--bg--highlight);

	@include mq($from: desktop) {
		display: block;
	}
}

.fret-range {
	@include space-children-v(20px);

	padding: 28px 20px;

	@include mq($from: desktop) {
		@include space-children-v(8px);

		padding: 14px;
	}
}

.fret-range__slider {
	min-width: 250px;
}

.fret-range__text {
	display: flex;
	align-items: center;
	justify-content: center;
	@include space-children-h(6px);

	color: var(--color--text);
}

.fret-range__text__separator {
	color: var(--color--text--secondary);
}

.export-menu {
	@include space-children-v(20px);

	padding: 10px;
}

.export-menu__text {
	@include space-children-v(10px);

	max-width: 300px;

	text-align: justify;
	font-size: 1.6rem;

	color: var(--color--text);
}

.export-menu__buttons {
	display: flex;
	justify-content: center;
	@include space-children-h(10px);
}

</style>
<!--}}}-->
