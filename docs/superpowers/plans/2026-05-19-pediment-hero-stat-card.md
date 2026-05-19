# Pediment Hero — Stat-Card Variant (Plan 5)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans. Steps use checkbox (`- [ ]`).

**Goal:** Add a `stat-card` variant to `starter/hero` matching the locked mockup hero: two-column split — left = eyebrow chip + headline + lead + primary & secondary pill buttons + trust ticks; right = a photo card with a frosted-glass stat overlay (big value + sub-line + 3-metric row). Existing `default`/`centered`/`media-bg` variants stay byte-identical.

**Architecture:** Extend the existing block — add one enum value (`stat-card`) + new attributes (`eyebrow`, `secondaryText`, `secondaryUrl`, `ticks[]`, `statValue`, `statText`, `metrics[]`; reuse existing `mediaId` for the photo). `render.php` branches: `if 'stat-card' === $variant` → new split markup; **else → the existing markup, unchanged byte-for-byte** (so the 3 behavioral HeroTest cases keep passing). `edit.tsx` gains the variant option + InspectorControls for the new structured fields. `style.scss` appends a `.is-variant-stat-card` block (existing rules untouched). The pre-existing `HeroTest` anti-phantom-variant guard (`test_block_json_variant_enum_excludes_split`) is updated to the new exact enum **and** keeps its intent by additionally asserting `stat-card` actually renders distinct markup.

**Tech Stack:** WordPress FSE block theme, apiVersion 3, `@wordpress/scripts` (TS/SCSS), PHP 8.1, PHPUnit (wp-env).

**Spec:** `docs/superpowers/specs/2026-05-17-pediment-design-system-design.md` (hero photo + frosted-stats overlay). Visual ref: `docs/design/pediment-mockup.html` (the `.hero` / `.hero-fig` / `.glass` section). Builds on merged Plans 1–4 (tokens, `.chip` utility, `--r-*`/`--section`, `accent-tint`).

**Scope:** ONLY `src/blocks/hero/*` + `tests/phpunit/BlockRender/HeroTest.php`. NOT here: pull-quote→testimonial (Plan 6), blog-index→Insights (Plan 7). No new JS/view-script. Do not touch other blocks, parts, theme.json, registration, or any `mega-*`.

**Pre-existing test contract (must honor):** `HeroTest::test_block_json_variant_enum_excludes_split` asserts `enum === ['default','centered','media-bg']` (guard: no advertised variant the renderer doesn't produce). Adding `stat-card` requires updating this test to the new exact array **and** adding a render assertion proving `stat-card` produces distinct output (keeps the original intent). `test_block_json_description_does_not_mention_split` must keep passing — the new description must mention the stat-card variant but NEVER the word "split". The 3 behavioral tests (`renders_headline_and_subheadline`, `renders_variant_class`, `omits_cta_when_url_is_empty`) must stay green ⇒ the non-stat-card render path stays byte-identical.

**Verification constraint:** Worktree NOT mounted in wp-env. Per task: env-independent gates — `npm run build` (compiles TS/SCSS), `php -l render.php`, valid `block.json` JSON, SCSS brace-balance, and a static trace of every HeroTest method against the render.php you ship. Full PHPUnit runs POST-MERGE in `:8888`/`:8889`. **Definition of done: post-merge PHPUnit green (all HeroTest incl. new stat-card cases + the rest of the suite); `npm run build` clean.**

---

## File Structure

| File | Action |
|---|---|
| `src/blocks/hero/block.json` | Modify — enum + new attributes + description |
| `tests/phpunit/BlockRender/HeroTest.php` | Modify — update enum guard, keep 3 behavioral, add stat-card cases |
| `src/blocks/hero/render.php` | Modify — add `stat-card` branch; else byte-identical |
| `src/blocks/hero/edit.tsx` | Modify — variant option + inspector fields |
| `src/blocks/hero/style.scss` | Modify — append `.is-variant-stat-card` rules |
| `src/blocks/hero/index.tsx` | Unchanged |

Each task commits.

---

### Task 1: block.json attributes + HeroTest contract

**Files:** Modify `src/blocks/hero/block.json`; Modify `tests/phpunit/BlockRender/HeroTest.php`.

- [ ] **Step 1: Replace `src/blocks/hero/block.json` with EXACTLY:**
```json
{
	"$schema": "https://schemas.wp.org/trunk/block.json",
	"apiVersion": 3,
	"name": "starter/hero",
	"title": "Hero",
	"category": "starter",
	"description": "A page-opening hero with headline, subheadline, and primary CTA. Variants: default, centered, media-bg, stat-card.",
	"textdomain": "starter",
	"supports": { "html": false, "align": [ "wide", "full" ] },
	"attributes": {
		"variant": {
			"type": "string",
			"default": "default",
			"enum": [ "default", "centered", "media-bg", "stat-card" ]
		},
		"headline": { "type": "string", "default": "" },
		"subheadline": { "type": "string", "default": "" },
		"ctaText": { "type": "string", "default": "" },
		"ctaUrl": { "type": "string", "default": "" },
		"secondaryText": { "type": "string", "default": "" },
		"secondaryUrl": { "type": "string", "default": "" },
		"eyebrow": { "type": "string", "default": "" },
		"ticks": { "type": "array", "default": [] },
		"statValue": { "type": "string", "default": "" },
		"statText": { "type": "string", "default": "" },
		"metrics": { "type": "array", "default": [] },
		"mediaId": { "type": "number", "default": 0 }
	},
	"editorScript": "file:./index.js",
	"editorStyle": "file:./style-index.css",
  "style": "file:./style-index.css",
	"render": "file:./render.php"
}
```
(Indentation/quoting matches the existing file: tabs, the two-space-indented `"style"` line preserved exactly as in the original.)

- [ ] **Step 2: Update `tests/phpunit/BlockRender/HeroTest.php`** — keep the `render()` helper and the 3 behavioral tests (`test_renders_headline_and_subheadline`, `test_renders_variant_class`, `test_omits_cta_when_url_is_empty`) EXACTLY as they are. Replace the method `test_block_json_variant_enum_excludes_split` with the renamed/updated guard below, KEEP `test_block_json_description_does_not_mention_split` as-is, and ADD the four new stat-card tests. The full new tail of the class (everything from `test_block_json_variant_enum_excludes_split` onward) becomes EXACTLY:
```php
	public function test_block_json_variant_enum_is_exact_and_renderable() {
		$path = dirname( __DIR__, 3 ) . '/src/blocks/hero/block.json';
		$this->assertFileIsReadable( $path );
		$data = json_decode( file_get_contents( $path ), true );
		$this->assertIsArray( $data );
		$this->assertSame(
			array( 'default', 'centered', 'media-bg', 'stat-card' ),
			$data['attributes']['variant']['enum'],
			'block.json variant enum must list exactly the variants the renderer produces'
		);
		// Anti-phantom guard: the newly advertised variant must render distinctly.
		$html = $this->render( array( 'variant' => 'stat-card', 'headline' => 'X' ) );
		$this->assertStringContainsString( 'is-variant-stat-card', $html );
	}

	public function test_block_json_description_does_not_mention_split() {
		$path = dirname( __DIR__, 3 ) . '/src/blocks/hero/block.json';
		$this->assertFileIsReadable( $path );
		$data = json_decode( file_get_contents( $path ), true );
		$this->assertIsArray( $data );
		$this->assertStringNotContainsStringIgnoringCase( 'split', $data['description'] );
	}

	public function test_stat_card_renders_eyebrow_secondary_and_ticks() {
		$html = $this->render(
			array(
				'variant'       => 'stat-card',
				'headline'      => 'We help leaders',
				'subheadline'   => 'Senior-led work.',
				'eyebrow'       => 'Strategy Consulting',
				'ctaText'       => 'Start',
				'ctaUrl'        => '/start',
				'secondaryText' => 'Our work',
				'secondaryUrl'  => '/work',
				'ticks'         => array( '120+ engagements', 'Global delivery' ),
			)
		);
		$this->assertStringContainsString( 'starter-hero__eyebrow', $html );
		$this->assertStringContainsString( 'Strategy Consulting', $html );
		$this->assertStringContainsString( 'href="/start"', $html );
		$this->assertStringContainsString( 'href="/work"', $html );
		$this->assertStringContainsString( 'Our work', $html );
		$this->assertStringContainsString( 'starter-hero__tick', $html );
		$this->assertStringContainsString( '120+ engagements', $html );
		$this->assertStringContainsString( 'Global delivery', $html );
	}

	public function test_stat_card_renders_glass_stat_and_metrics() {
		$html = $this->render(
			array(
				'variant'   => 'stat-card',
				'headline'  => 'H',
				'statValue' => '+34%',
				'statText'  => 'margin improvement',
				'metrics'   => array(
					array( 'value' => '18', 'label' => 'countries' ),
					array( 'value' => '94%', 'label' => 'repeat clients' ),
				),
			)
		);
		$this->assertStringContainsString( 'starter-hero__glass', $html );
		$this->assertStringContainsString( '+34%', $html );
		$this->assertStringContainsString( 'margin improvement', $html );
		$this->assertStringContainsString( 'starter-hero__metric', $html );
		$this->assertStringContainsString( '18', $html );
		$this->assertStringContainsString( 'countries', $html );
		$this->assertStringContainsString( '94%', $html );
		$this->assertStringContainsString( 'repeat clients', $html );
	}

	public function test_stat_card_omits_secondary_when_url_missing() {
		$html = $this->render(
			array(
				'variant'       => 'stat-card',
				'headline'      => 'H',
				'secondaryText' => 'Our work',
				'secondaryUrl'  => '',
			)
		);
		$this->assertStringNotContainsString( 'starter-hero__cta--secondary', $html );
	}

	public function test_default_variant_markup_unchanged() {
		$html = $this->render(
			array( 'variant' => 'default', 'headline' => 'D', 'subheadline' => 'S' )
		);
		$this->assertStringContainsString( 'is-variant-default', $html );
		$this->assertStringContainsString( 'starter-hero__headline', $html );
		$this->assertStringNotContainsString( 'starter-hero__glass', $html );
		$this->assertStringNotContainsString( 'starter-hero__eyebrow', $html );
	}
```

- [ ] **Step 3: Verify (env-independent)** — `python3 -c "import json;json.load(open('src/blocks/hero/block.json'));print('JSON-OK')"`; confirm enum is exactly the 4 values and description contains "stat-card" and NOT "split"; `php -l tests/phpunit/BlockRender/HeroTest.php`; `npm run build` still compiles. `git diff` touches only those 2 files.

- [ ] **Step 4: Commit**
```bash
git add src/blocks/hero/block.json tests/phpunit/BlockRender/HeroTest.php
git commit -m "feat(hero): block.json stat-card variant + HeroTest contract"
```

---

### Task 2: render.php — stat-card branch

**Files:** Modify `src/blocks/hero/render.php`.

The existing variable extraction + the default/centered/media-bg markup must stay byte-identical (wrap it in an `else`). Add a `stat-card` branch above it.

- [ ] **Step 1: Replace `src/blocks/hero/render.php` with EXACTLY:**
```php
<?php
/**
 * Server-side render for starter/hero.
 *
 * @var array $attributes
 */

$variant     = isset( $attributes['variant'] ) ? (string) $attributes['variant'] : 'default';
$headline    = isset( $attributes['headline'] ) ? (string) $attributes['headline'] : '';
$subheadline = isset( $attributes['subheadline'] ) ? (string) $attributes['subheadline'] : '';
$cta_text    = isset( $attributes['ctaText'] ) ? (string) $attributes['ctaText'] : '';
$cta_url     = isset( $attributes['ctaUrl'] ) ? (string) $attributes['ctaUrl'] : '';
$media_id    = isset( $attributes['mediaId'] ) ? (int) $attributes['mediaId'] : 0;

$wrapper_attributes = get_block_wrapper_attributes(
	array(
		'class' => 'starter-hero is-variant-' . sanitize_html_class( $variant ),
	)
);

if ( 'stat-card' === $variant ) {
	$eyebrow     = isset( $attributes['eyebrow'] ) ? (string) $attributes['eyebrow'] : '';
	$sec_text    = isset( $attributes['secondaryText'] ) ? (string) $attributes['secondaryText'] : '';
	$sec_url     = isset( $attributes['secondaryUrl'] ) ? (string) $attributes['secondaryUrl'] : '';
	$stat_value  = isset( $attributes['statValue'] ) ? (string) $attributes['statValue'] : '';
	$stat_text   = isset( $attributes['statText'] ) ? (string) $attributes['statText'] : '';
	$ticks       = ( isset( $attributes['ticks'] ) && is_array( $attributes['ticks'] ) ) ? $attributes['ticks'] : array();
	$metrics     = ( isset( $attributes['metrics'] ) && is_array( $attributes['metrics'] ) ) ? $attributes['metrics'] : array();

	ob_start();
	?>
	<section <?php echo $wrapper_attributes; // phpcs:ignore WordPress.Security.EscapeOutput ?>>
		<div class="starter-hero__col">
			<?php if ( '' !== $eyebrow ) : ?>
				<span class="starter-hero__eyebrow"><?php echo wp_kses_post( $eyebrow ); ?></span>
			<?php endif; ?>
			<?php if ( '' !== $headline ) : ?>
				<h1 class="starter-hero__headline"><?php echo wp_kses_post( $headline ); ?></h1>
			<?php endif; ?>
			<?php if ( '' !== $subheadline ) : ?>
				<p class="starter-hero__subheadline"><?php echo wp_kses_post( $subheadline ); ?></p>
			<?php endif; ?>
			<div class="starter-hero__actions">
				<?php if ( '' !== $cta_text && '' !== $cta_url ) : ?>
					<a class="starter-hero__cta" href="<?php echo esc_url( $cta_url ); ?>"><?php echo wp_kses_post( $cta_text ); ?></a>
				<?php endif; ?>
				<?php if ( '' !== $sec_text && '' !== $sec_url ) : ?>
					<a class="starter-hero__cta starter-hero__cta--secondary" href="<?php echo esc_url( $sec_url ); ?>"><?php echo wp_kses_post( $sec_text ); ?></a>
				<?php endif; ?>
			</div>
			<?php if ( ! empty( $ticks ) ) : ?>
				<ul class="starter-hero__ticks">
					<?php foreach ( $ticks as $tick ) : ?>
						<li class="starter-hero__tick"><?php echo wp_kses_post( (string) $tick ); ?></li>
					<?php endforeach; ?>
				</ul>
			<?php endif; ?>
		</div>
		<figure class="starter-hero__fig">
			<?php
			if ( $media_id ) {
				echo wp_get_attachment_image( $media_id, 'large', false, array( 'class' => 'starter-hero__img' ) ); // phpcs:ignore WordPress.Security.EscapeOutput
			}
			?>
			<?php if ( '' !== $stat_value || '' !== $stat_text || ! empty( $metrics ) ) : ?>
				<div class="starter-hero__glass">
					<?php if ( '' !== $stat_value ) : ?>
						<div class="starter-hero__stat-value"><?php echo wp_kses_post( $stat_value ); ?></div>
					<?php endif; ?>
					<?php if ( '' !== $stat_text ) : ?>
						<div class="starter-hero__stat-text"><?php echo wp_kses_post( $stat_text ); ?></div>
					<?php endif; ?>
					<?php if ( ! empty( $metrics ) ) : ?>
						<div class="starter-hero__metrics">
							<?php foreach ( $metrics as $m ) : ?>
								<?php
								$mv = is_array( $m ) && isset( $m['value'] ) ? (string) $m['value'] : '';
								$ml = is_array( $m ) && isset( $m['label'] ) ? (string) $m['label'] : '';
								?>
								<div class="starter-hero__metric">
									<b><?php echo wp_kses_post( $mv ); ?></b>
									<span><?php echo wp_kses_post( $ml ); ?></span>
								</div>
							<?php endforeach; ?>
						</div>
					<?php endif; ?>
				</div>
			<?php endif; ?>
		</figure>
	</section>
	<?php
	echo ob_get_clean();
	return;
}

$bg_style = '';
if ( 'media-bg' === $variant && $media_id ) {
	$url = wp_get_attachment_image_url( $media_id, 'full' );
	if ( $url ) {
		$bg_style = ' style="background-image:url(' . esc_url( $url ) . ');"';
	}
}

ob_start();
?>
<section <?php echo $wrapper_attributes; // phpcs:ignore WordPress.Security.EscapeOutput ?><?php echo $bg_style; // phpcs:ignore WordPress.Security.EscapeOutput ?>>
	<?php if ( $headline ) : ?>
		<h1 class="starter-hero__headline"><?php echo wp_kses_post( $headline ); ?></h1>
	<?php endif; ?>
	<?php if ( $subheadline ) : ?>
		<p class="starter-hero__subheadline"><?php echo wp_kses_post( $subheadline ); ?></p>
	<?php endif; ?>
	<?php if ( $cta_text && $cta_url ) : ?>
		<a class="starter-hero__cta" href="<?php echo esc_url( $cta_url ); ?>">
			<?php echo wp_kses_post( $cta_text ); ?>
		</a>
	<?php endif; ?>
</section>
<?php
echo ob_get_clean();
```
(The block from `$bg_style = '';` onward is the original render verbatim — the only change is the inserted `if ( 'stat-card' === $variant ) { … return; }` branch above it, plus moving the new attribute reads inside that branch.)

- [ ] **Step 2: Verify** — `php -l src/blocks/hero/render.php`; `npm run build`. Static-trace ALL HeroTest methods: the 3 behavioral + `test_default_variant_markup_unchanged` hit the else branch (markup identical to original ⇒ pass; no `__glass`/`__eyebrow`); `test_block_json_variant_enum_is_exact_and_renderable` stat-card render contains `is-variant-stat-card` (wrapper) ⇒ pass; `test_stat_card_renders_eyebrow_secondary_and_ticks` / `..._glass_stat_and_metrics` / `..._omits_secondary_when_url_missing` map to the new branch's classes/guards. Confirm each assertion. Only `render.php` changed.

- [ ] **Step 3: Commit**
```bash
git add src/blocks/hero/render.php
git commit -m "feat(hero): stat-card render branch (split + glass stats)"
```

---

### Task 3: edit.tsx — variant option + inspector fields

**Files:** Modify `src/blocks/hero/edit.tsx`.

- [ ] **Step 1: Replace `src/blocks/hero/edit.tsx` with EXACTLY:**
```tsx
import { __ } from '@wordpress/i18n';
import {
	useBlockProps,
	RichText,
	InspectorControls,
	MediaUpload,
} from '@wordpress/block-editor';
import {
	PanelBody,
	SelectControl,
	TextControl,
	TextareaControl,
	Button,
} from '@wordpress/components';

type Metric = { value: string; label: string };
type Attrs = {
	variant: 'default' | 'centered' | 'media-bg' | 'stat-card';
	headline: string;
	subheadline: string;
	ctaText: string;
	ctaUrl: string;
	secondaryText: string;
	secondaryUrl: string;
	eyebrow: string;
	ticks: string[];
	statValue: string;
	statText: string;
	metrics: Metric[];
	mediaId: number;
};

export default function Edit( {
	attributes,
	setAttributes,
}: {
	attributes: Attrs;
	setAttributes: ( a: Partial< Attrs > ) => void;
} ) {
	const blockProps = useBlockProps( {
		className: `starter-hero is-variant-${ attributes.variant }`,
	} );
	const isStatCard = attributes.variant === 'stat-card';

	return (
		<>
			<InspectorControls>
				<PanelBody title={ __( 'Hero settings', 'starter' ) }>
					<SelectControl
						label={ __( 'Variant', 'starter' ) }
						value={ attributes.variant }
						options={ [
							{ label: 'Default', value: 'default' },
							{ label: 'Centered', value: 'centered' },
							{ label: 'Media BG', value: 'media-bg' },
							{ label: 'Stat card', value: 'stat-card' },
						] }
						onChange={ ( v ) =>
							setAttributes( {
								variant: v as Attrs[ 'variant' ],
							} )
						}
					/>
					<TextControl
						label={ __( 'CTA URL', 'starter' ) }
						value={ attributes.ctaUrl }
						onChange={ ( v ) => setAttributes( { ctaUrl: v } ) }
					/>
					{ isStatCard && (
						<>
							<TextControl
								label={ __( 'Eyebrow', 'starter' ) }
								value={ attributes.eyebrow }
								onChange={ ( v ) =>
									setAttributes( { eyebrow: v } )
								}
							/>
							<TextControl
								label={ __( 'Secondary CTA text', 'starter' ) }
								value={ attributes.secondaryText }
								onChange={ ( v ) =>
									setAttributes( { secondaryText: v } )
								}
							/>
							<TextControl
								label={ __( 'Secondary CTA URL', 'starter' ) }
								value={ attributes.secondaryUrl }
								onChange={ ( v ) =>
									setAttributes( { secondaryUrl: v } )
								}
							/>
							<TextareaControl
								label={ __(
									'Trust ticks (one per line)',
									'starter'
								) }
								value={ ( attributes.ticks || [] ).join(
									'\n'
								) }
								onChange={ ( v ) =>
									setAttributes( {
										ticks: v
											.split( '\n' )
											.map( ( s ) => s.trim() )
											.filter( Boolean ),
									} )
								}
							/>
							<TextControl
								label={ __( 'Stat value', 'starter' ) }
								value={ attributes.statValue }
								onChange={ ( v ) =>
									setAttributes( { statValue: v } )
								}
							/>
							<TextControl
								label={ __( 'Stat text', 'starter' ) }
								value={ attributes.statText }
								onChange={ ( v ) =>
									setAttributes( { statText: v } )
								}
							/>
							<TextareaControl
								label={ __(
									'Metrics — “value | label” per line',
									'starter'
								) }
								value={ ( attributes.metrics || [] )
									.map(
										( m ) => `${ m.value } | ${ m.label }`
									)
									.join( '\n' ) }
								onChange={ ( v ) =>
									setAttributes( {
										metrics: v
											.split( '\n' )
											.map( ( line ) => {
												const [ value, label ] =
													line.split( '|' );
												return {
													value: ( value || '' ).trim(),
													label: ( label || '' ).trim(),
												};
											} )
											.filter(
												( m ) =>
													m.value !== '' ||
													m.label !== ''
											),
									} )
								}
							/>
						</>
					) }
					{ ( attributes.variant === 'media-bg' ||
						isStatCard ) && (
						<MediaUpload
							allowedTypes={ [ 'image' ] }
							onSelect={ ( media: any ) =>
								setAttributes( { mediaId: media.id } )
							}
							render={ ( { open }: { open: () => void } ) => (
								<Button variant="secondary" onClick={ open }>
									{ attributes.mediaId
										? __( 'Replace image', 'starter' )
										: __( 'Pick image', 'starter' ) }
								</Button>
							) }
						/>
					) }
				</PanelBody>
			</InspectorControls>

			<div { ...blockProps }>
				{ isStatCard && (
					<RichText
						tagName="span"
						className="starter-hero__eyebrow"
						value={ attributes.eyebrow }
						onChange={ ( v ) =>
							setAttributes( { eyebrow: v } )
						}
						placeholder={ __( 'Eyebrow…', 'starter' ) }
					/>
				) }
				<RichText
					tagName="h1"
					className="starter-hero__headline"
					value={ attributes.headline }
					onChange={ ( v ) => setAttributes( { headline: v } ) }
					placeholder={ __( 'Headline…', 'starter' ) }
				/>
				<RichText
					tagName="p"
					className="starter-hero__subheadline"
					value={ attributes.subheadline }
					onChange={ ( v ) =>
						setAttributes( { subheadline: v } )
					}
					placeholder={ __( 'Subheadline…', 'starter' ) }
				/>
				<RichText
					tagName="span"
					className="starter-hero__cta"
					value={ attributes.ctaText }
					onChange={ ( v ) => setAttributes( { ctaText: v } ) }
					placeholder={ __( 'CTA text…', 'starter' ) }
				/>
			</div>
		</>
	);
}
```

- [ ] **Step 2: Verify** — `npm run build` compiles (authoritative TS gate via ts-loader; do NOT rely on standalone `npx tsc`, which mis-fires without the project tsconfig — only flag a TS issue if `npm run build` fails). Only `edit.tsx` changed.

- [ ] **Step 3: Commit**
```bash
git add src/blocks/hero/edit.tsx
git commit -m "feat(hero): editor controls for stat-card variant"
```

---

### Task 4: style.scss — `.is-variant-stat-card`

**Files:** Modify `src/blocks/hero/style.scss` (APPEND only; existing rules untouched).

- [ ] **Step 1: Append to the END of `src/blocks/hero/style.scss`** (leading blank line, then EXACTLY):
```scss

/* stat-card variant: split text + photo-with-glass-stats */
.starter-hero.is-variant-stat-card {
  display: grid;
  grid-template-columns: 1.05fr .95fr;
  gap: clamp(32px, 5vw, 60px);
  align-items: center;
  padding-block: clamp(56px, 7vw, 96px);

  .starter-hero__eyebrow {
    display: inline-flex;
    align-items: center;
    gap: 9px;
    background: var(--wp--preset--color--accent-tint);
    color: var(--wp--preset--color--accent-hover);
    font-size: 13px;
    font-weight: 600;
    padding: 8px 16px;
    border-radius: var(--r-pill, 999px);
    margin-bottom: var(--wp--preset--spacing--30);
  }

  .starter-hero__subheadline { max-width: 38ch; }

  .starter-hero__actions {
    display: flex;
    gap: 14px;
    flex-wrap: wrap;
    margin-bottom: var(--wp--preset--spacing--40);
  }

  .starter-hero__cta--secondary {
    background: var(--wp--preset--color--surface);
    color: var(--wp--preset--color--text);
    border: 1.5px solid var(--wp--preset--color--border);
  }
  .starter-hero__cta--secondary:hover {
    background: var(--wp--preset--color--surface);
    color: var(--wp--preset--color--accent);
    border-color: var(--wp--preset--color--accent);
  }

  .starter-hero__ticks {
    list-style: none;
    margin: 0;
    padding: 0;
    display: flex;
    flex-wrap: wrap;
    gap: 14px 30px;
  }
  .starter-hero__tick {
    color: var(--wp--preset--color--text-muted);
    font-size: .92rem;
    font-weight: 600;
  }

  .starter-hero__fig {
    position: relative;
    margin: 0;
    border-radius: var(--r-lg, 20px);
    overflow: hidden;
    box-shadow: var(--wp--preset--shadow--medium);
    aspect-ratio: 5 / 6;
    background: var(--wp--preset--color--primary);
  }
  .starter-hero__img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    display: block;
  }

  .starter-hero__glass {
    position: absolute;
    left: 18px;
    right: 18px;
    bottom: 18px;
    background: color-mix(in srgb, var(--wp--preset--color--primary) 60%, transparent);
    backdrop-filter: blur(10px);
    -webkit-backdrop-filter: blur(10px);
    border: 1px solid rgba(255, 255, 255, .16);
    border-radius: 16px;
    padding: 22px 24px;
    color: #fff;
  }
  .starter-hero__stat-value {
    font-size: 2.4rem;
    font-weight: 800;
    letter-spacing: -.02em;
    line-height: 1;
  }
  .starter-hero__stat-text {
    color: #c9d6ec;
    font-size: .88rem;
    margin-top: 6px;
  }
  .starter-hero__metrics {
    display: flex;
    gap: 22px;
    margin-top: 18px;
    border-top: 1px solid rgba(255, 255, 255, .16);
    padding-top: 16px;
  }
  .starter-hero__metric b {
    display: block;
    font-size: 1.15rem;
    font-weight: 800;
  }
  .starter-hero__metric span {
    font-size: .74rem;
    color: #c9d6ec;
  }

  @media (max-width: 781px) {
    grid-template-columns: 1fr;
  }
}
```

- [ ] **Step 2: Verify** — `npm run build` compiles; SCSS brace-balanced (`node -e` count); `git diff` is append-only to `style.scss` (zero `-` lines; existing default/centered/media-bg rules byte-unchanged). Only `style.scss` changed.

- [ ] **Step 3: Commit**
```bash
git add src/blocks/hero/style.scss
git commit -m "style(hero): stat-card variant (split + frosted glass stats)"
```

---

### Task 5: Build + cumulative guard

**Files:** none (verification only).

- [ ] **Step 1:** `npm run build` — compiles; `build/blocks/hero/{block.json,index.js,style-index.css,render.php}` present.
- [ ] **Step 2:** `git diff <branch-base>..HEAD --name-only` — ONLY `src/blocks/hero/{block.json,render.php,edit.tsx,style.scss}` + `tests/phpunit/BlockRender/HeroTest.php`. NO other blocks/parts/theme.json/inc/mega-*.
- [ ] **Step 3:** `php -l src/blocks/hero/render.php`; `python3` JSON-validates block.json (enum exactly 4, description has "stat-card" not "split"); SCSS brace-balanced; `git diff` confirms render.php's post-`stat-card` section (from `$bg_style = '';`) byte-identical to the pre-Plan-5 original. `git status --porcelain` clean (besides pre-existing untracked `docs/images/`).

**Post-merge (main checkout `:8888`/`:8889`, controller — NOT a worktree step):** `npm run build` → `npx wp-env run cli wp theme activate wp-starter-theme` → full `vendor/bin/phpunit` (expect: all HeroTest green incl. the updated enum guard + 4 new stat-card cases + `test_default_variant_markup_unchanged`; rest of suite unchanged). Playwright unaffected (no e2e changes; the unrelated mega-menu failures stay out of scope).

---

## Self-Review

**Spec coverage:** hero photo + frosted-stats overlay delivered as a `stat-card` variant — block.json attrs + HeroTest contract (Task 1), render branch (Task 2), editor controls (Task 3), styles (Task 4), guard (Task 5). pull-quote→testimonial and blog-index→Insights are explicitly Plans 6 & 7 — not gaps.

**Placeholder scan:** none — every block.json/php/tsx/scss/test block is complete and final.

**Type/name consistency:** New attrs (`eyebrow`,`secondaryText`,`secondaryUrl`,`ticks`,`statValue`,`statText`,`metrics`,`mediaId`) match across block.json ⇄ render.php reads ⇄ edit.tsx `Attrs` type/controls. Render BEM (`starter-hero__eyebrow/__col/__actions/__cta--secondary/__ticks/__tick/__fig/__img/__glass/__stat-value/__stat-text/__metrics/__metric`) match the appended SCSS selectors and the HeroTest assertions exactly. Enum `['default','centered','media-bg','stat-card']` identical in block.json and the updated `test_block_json_variant_enum_is_exact_and_renderable`. The non-stat-card render path (from `$bg_style`) is the verbatim original ⇒ the 3 behavioral tests + `test_default_variant_markup_unchanged` pass. `accent-tint`/`--r-pill`/`--r-lg`/`shadow--medium` are merged Plan-1/2 tokens; `color-mix` matches the Plan-3 frosted-header precedent.

**Regression safety:** Only hero files + HeroTest changed (Task 5 Step 2 asserts) ⇒ all other blocks’ PHPUnit + e2e untouched. `test_block_json_variant_enum_is_exact_and_renderable` preserves the original anti-phantom intent (enum must equal what the renderer produces — now additionally proven by a live `stat-card` render assertion). Description keeps clear of "split" so `test_block_json_description_does_not_mention_split` stays green. SCSS is append-only so existing variants are visually unchanged.
