<script context="module">
	import { engine } from '$lib/engine';
	import { formatValue, reduceAST } from 'publicodes';
	import Emoji from './Emoji.svelte';

	// In case the formula involve a “linear operation” such as additions or
	// product, we cannot extract a fixed set a threshold and we need to display
	// a input of type number.
	const numberFieldRequired = Symbol('number field required');

	// We do a static analysis of the rules AST to search for a particular rule name.
	// When in find it in a comparaison expression we retreive the value of the other
	// side of the expression.
	// For exemple in: applicable si: my . rule <= 400 €
	// We will retrieve [400]
	// We also need to handle mechanisms that are implemented using comparaisons such
	// as “grille” or “barème”.
	function findAllComparaisonsValue(dottedName, { searchedName, unit }) {
		const comparaisonOperators = ['<', '<=', '>', '>='];
		const linearOperatiors = ['-', '+', '*', '/'];

		// TODO: There might be a better way to convert a parsed node into a given unit.
		const convertValue = (node) => {
			const valeur = formatValue(engine.evaluate(node)).replace(/\s/g, '');
			if (valeur === '∞') {
				return Infinity;
			}
			return engine.evaluate({ valeur, unité: unit }).nodeValue;
		};

		const namesToFollow = ['ménage imposable', 'personnes dans le foyer fiscal'];

		const accumulateThresholds = (acc, node) => {
			if (acc === numberFieldRequired) {
				return numberFieldRequired;
			}

			if (node.nodeKind === 'grille' && node.explanation.assiette?.dottedName === searchedName) {
				const thresholds = node.explanation.tranches.map(({ plafond }) => convertValue(plafond));
				return [...acc, ...thresholds];
			} else if (
				node.nodeKind === 'operation' &&
				comparaisonOperators.includes(node.operationKind)
			) {
				if (node.explanation[0]?.dottedName === searchedName) {
					return [...acc, convertValue(node.explanation[1])];
				} else if (node.explanation[1]?.dottedName === searchedName) {
					return [...acc, convertValue(node.explanation[0])];
				}
			} else if (
				node.nodeKind === 'operation' &&
				linearOperatiors.includes(node.operationKind) &&
				(node.explanation[0]?.dottedName === searchedName ||
					node.explanation[1]?.dottedName === searchedName)
			) {
				return numberFieldRequired;
			}
			// Following reference like this is fragile. We could follow all references
			// but would need some special handling for the dependency on parent.
			// For now this works well enough.
			else if (node.nodeKind === 'reference' && namesToFollow.includes(node.name)) {
				return [...acc, ...findAllComparaisonsValue(node.dottedName, { searchedName, unit })];
			}
		};

		return reduceAST(accumulateThresholds, [], engine.getRule(dottedName));
	}
</script>

<script>
	import { answers, publicodeSituation } from '$lib/stores';
	import { slide } from 'svelte/transition';
	import MultipleChoiceAnswer from './MultipleChoiceAnswer.svelte';
	import NumberField from './NumberField.svelte';
	import { page } from '$app/stores';

	// TODO Hacky
	const veloCat = $page.url.searchParams.get('velo');

	export let goals;
	let value;

	const uniq = (l) => [...new Set(l)];
	const tresholds = goals.flatMap((name) =>
		findAllComparaisonsValue(name, {
			searchedName: 'revenu fiscal de référence',
			unit: '€/mois'
		})
	);

	const numberFieldIsRequired = tresholds.includes(numberFieldRequired);

	// Not all statically detected thresholds are impactful for the current computation
	// as some of them might be in inactive branches of the computation.
	// We evaluate all the thresholds and only keep the one that induice a change in the result.
	const engineBis = engine.shallowCopy();
	$: displayedThresholds =
		!numberFieldIsRequired &&
		[
			0,
			...uniq(
				tresholds

					.filter((x) => x !== Infinity)
					.map((x) => Math.round(x))
					.sort((a, b) => a - b)
			)
		]
			.reduce(
				(acc, revenu) => {
					// HACK: The problem with thresholds evaluation is that their
					// value might depend on other answers, for exemple the bike
					// price. This implies that in some case we remove relevant
					// thresholds. This should be fixed with a more clever static
					// analysis.
					//
					// The line below is used to reactively recompute thresholds
					// when a answer is given.
					$answers;
					engineBis.setSituation({
						...$publicodeSituation,
						'vélo . type': `'${veloCat}'`,
						'maximiser les aides': 'oui',
						'revenu fiscal de référence': `${revenu + 1} €/mois`
					});
					const montantAides = engineBis.evaluate('aides . montant').nodeValue;
					if (montantAides === acc.dernierMontant) {
						return acc;
					} else {
						return {
							thresholds: [...acc.thresholds, revenu],
							dernierMontant: montantAides
						};
					}
				},
				{ thresholds: [], dernierMontant: null }
			)
			.thresholds.slice(1);

	let showExplanations = false;

	$: $answers['revenu fiscal de référence'] = value && `${value} €/mois`;
</script>

{#if numberFieldIsRequired || displayedThresholds.length > 0}
	<div class="mt-6">
		Quel est votre revenu net mensuel (quotient familial) ? <span
			title="Plus d’informations"
			class="cursor-pointer"
			on:click={() => (showExplanations = !showExplanations)}><Emoji emoji="ℹ" /></span
		>
		{#if showExplanations}
			<p class="m-4 mt-2 text-gray-600 text-sm" transition:slide|local={{ duration: 100 }}>
				Le montant des aides dépend de votre revenu par part de quotient familial. Sur votre avis
				d'imposition cela correspond au montant du « revenu fiscal de référence » divisé par le
				nombre de parts du quotient familial, puis divisé par 12.
			</p>
		{/if}
		<div class="flex gap-2 mt-2 flex-wrap">
			{#if numberFieldIsRequired}
				<NumberField bind:value id="revenu-fiscal" unité="€" />
			{:else}
				{#each displayedThresholds as threshold}
					<MultipleChoiceAnswer value={threshold - 1} bind:group={value}
						>moins de <strong class="font-semibold">{formatValue(threshold)} €</strong
						></MultipleChoiceAnswer
					>
				{/each}
				<MultipleChoiceAnswer
					value={displayedThresholds[displayedThresholds.length - 1] + 1}
					bind:group={value}
					>plus de <strong class="font-semibold"
						>{formatValue(displayedThresholds[displayedThresholds.length - 1] + 1)} €</strong
					></MultipleChoiceAnswer
				>
			{/if}
		</div>
	</div>
{/if}
