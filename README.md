# WordPress Subsection Pattern: Custom Post Type with Homepage Selection

This document outlines a pattern for creating "subsections" in WordPress using custom post types with configurable homepage selection. This approach allows you to create dedicated sections of your site (like `/about`, `/services`, `/resources`) that behave like mini-sites within your main WordPress installation.

## Overview

The pattern consists of:
1. **Custom Post Type** with archive enabled
2. **Settings Page** for homepage selection
3. **Archive Template** that displays selected homepage content
4. **Admin Bar Integration** for easy editing

## Use Cases

- **Resource Centers**: `/resources` with a dedicated homepage
- **Service Areas**: `/services` with overview page
- **About Sections**: `/about` with main about page
- **Program Areas**: `/programs` with program overview
- **Department Pages**: `/departments` with department landing

## Implementation

### 1. Custom Post Type Registration

```php
<?php
/**
 * Register Custom Post Type for Subsection
 */
function register_subsection_post_type() {
    $labels = array(
        'name'                  => _x('Subsection Pages', 'Post type general name', 'theme'),
        'singular_name'         => _x('Subsection Page', 'Post type singular name', 'theme'),
        'menu_name'             => _x('Subsection Pages', 'Admin Menu text', 'theme'),
        'add_new'               => __('Add New', 'theme'),
        'add_new_item'          => __('Add New Subsection Page', 'theme'),
        'edit_item'             => __('Edit Subsection Page', 'theme'),
        'all_items'             => __('All Subsection Pages', 'theme'),
    );

    $args = array(
        'labels'             => $labels,
        'public'             => true,
        'publicly_queryable' => true,
        'show_ui'            => true,
        'show_in_menu'       => true,
        'query_var'          => true,
        'rewrite'            => array('slug' => 'subsection'), // Change this to your desired URL
        'capability_type'    => 'post',
        'has_archive'        => true, // Enable archive page
        'hierarchical'       => true, // Allow parent/child relationships
        'menu_position'      => 20,
        'menu_icon'          => 'dashicons-admin-page',
        'supports'           => array(
            'title',
            'editor',
            'thumbnail',
            'excerpt',
            'page-attributes',
            'custom-fields',
            'revisions'
        ),
        'show_in_rest'       => true, // Enable Gutenberg editor
    );

    register_post_type('subsection-page', $args);
}
add_action('init', 'register_subsection_post_type');
```

### 2. Settings Page for Homepage Selection

```php
<?php
/**
 * Subsection Settings Page
 */
function subsection_settings_init() {
    // Register settings
    register_setting('subsection_settings', 'subsection_homepage_id');
    
    // Add settings section
    add_settings_section(
        'subsection_general_settings',
        __('General Subsection Settings', 'theme'),
        'subsection_general_settings_callback',
        'subsection_settings'
    );
    
    // Add settings fields
    add_settings_field(
        'subsection_homepage_id',
        __('Subsection Homepage', 'theme'),
        'subsection_homepage_field_callback',
        'subsection_settings',
        'subsection_general_settings'
    );
}
add_action('admin_init', 'subsection_settings_init');

/**
 * Add Subsection Settings menu page
 */
function subsection_settings_menu() {
    add_submenu_page(
        'edit.php?post_type=subsection-page',
        __('Subsection Settings', 'theme'),
        __('Settings', 'theme'),
        'manage_options',
        'subsection-settings',
        'subsection_settings_page_callback'
    );
}
add_action('admin_menu', 'subsection_settings_menu');

/**
 * Subsection Settings page callback
 */
function subsection_settings_page_callback() {
    ?>
    <div class="wrap">
        <h1><?php echo esc_html(get_admin_page_title()); ?></h1>
        
        <form method="post" action="options.php">
            <?php
            settings_fields('subsection_settings');
            do_settings_sections('subsection_settings');
            submit_button();
            ?>
        </form>
    </div>
    <?php
}

/**
 * Subsection Homepage field callback
 */
function subsection_homepage_field_callback() {
    $current_homepage_id = get_option('subsection_homepage_id', '');
    
    // Get all subsection pages
    $subsection_pages = get_posts(array(
        'post_type' => 'subsection-page',
        'numberposts' => -1,
        'post_status' => 'publish',
        'orderby' => 'title',
        'order' => 'ASC'
    ));
    
    echo '<select name="subsection_homepage_id" id="subsection_homepage_id">';
    echo '<option value="">' . __('-- Select Subsection Homepage --', 'theme') . '</option>';
    
    foreach ($subsection_pages as $page) {
        $selected = selected($current_homepage_id, $page->ID, false);
        echo '<option value="' . esc_attr($page->ID) . '" ' . $selected . '>';
        echo esc_html($page->post_title);
        echo '</option>';
    }
    
    echo '</select>';
    
    if ($current_homepage_id) {
        $homepage_url = get_permalink($current_homepage_id);
        echo '<p class="description">';
        printf(
            __('Current homepage: <a href="%s" target="_blank">%s</a>', 'theme'),
            esc_url($homepage_url),
            esc_html(get_the_title($current_homepage_id))
        );
        echo '</p>';
    }
    
    echo '<p class="description">';
    _e('This page will be displayed when someone visits your subsection URL.', 'theme');
    echo '</p>';
}

/**
 * Helper functions
 */
function get_subsection_homepage_id() {
    return get_option('subsection_homepage_id', '');
}

function get_subsection_homepage_post() {
    $homepage_id = get_subsection_homepage_id();
    return $homepage_id ? get_post($homepage_id) : null;
}
```

### 3. Archive Template

Create `archive-subsection-page.php` in your theme:

```php
<?php
/**
 * Archive template for Subsection Pages
 * 
 * This template displays the selected subsection homepage content
 */

// Add admin bar edit link for the selected homepage
add_filter('wp_admin_bar_edit_menu_args', function($args) {
    if (is_post_type_archive('subsection-page')) {
        $subsection_homepage = get_subsection_homepage_post();
        if ($subsection_homepage) {
            $args['href'] = get_edit_post_link($subsection_homepage->ID);
        }
    }
    return $args;
});

get_header(); ?>

<div id="primary" class="content-area">
    <main id="main" class="site-main">
        
        <?php
        // Get the selected subsection homepage
        $subsection_homepage = get_subsection_homepage_post();
        
        if ($subsection_homepage) :
            // Set up post data for the homepage
            setup_postdata($subsection_homepage);
            ?>
            
            <article id="post-<?php echo $subsection_homepage->ID; ?>" <?php post_class('subsection-homepage-content'); ?>>
                
                <div class="entry-content">
                    <?php echo apply_filters('the_content', $subsection_homepage->post_content); ?>
                </div>

            </article>
            
            <?php
            // Reset post data
            wp_reset_postdata();
            
        else :
            ?>
            
            <header class="page-header">
                <h1 class="page-title">Subsection</h1>
            </header>
            
            <div class="entry-content">
                <p><?php _e('No subsection homepage has been selected. Please set a homepage in the Subsection Settings.', 'theme'); ?></p>
            </div>
            
        <?php endif; ?>

    </main>
</div>

<?php
get_sidebar();
get_footer();
```

### 4. Flush Rewrite Rules

```php
/**
 * Flush rewrite rules on theme activation
 */
function subsection_flush_rewrite_rules() {
    register_subsection_post_type();
    flush_rewrite_rules();
}
add_action('after_switch_theme', 'subsection_flush_rewrite_rules');
```

## GenerateBlocks Integration

When working with GenerateBlocks, this pattern works particularly well because:

### Location Rules
GenerateBlocks location rules can target:
- **Post Type Archive**: `Post Type Archive: Subsection Page`
- **Post Type**: `Post Type: Subsection Page`

### Header/Footer Application
Since the archive template uses standard WordPress functions (`get_header()`, `get_footer()`), GenerateBlocks can apply:
- Headers with location rule: "Post Type Archive: Subsection Page"
- Footers with location rule: "Post Type Archive: Subsection Page"
- Global styles and CSS

### Content Blocks
You can create GenerateBlocks content blocks with location rules:
- "Post Type Archive: Subsection Page" - for the main content area
- "Post Type: Subsection Page" - for individual subsection pages

## Customization Options

### Multiple Subsections
To create multiple subsections, duplicate the pattern with different post types:

```php
// For /about subsection
register_post_type('about-page', array(
    'rewrite' => array('slug' => 'about'),
    'has_archive' => true,
    // ... other args
));

// For /services subsection  
register_post_type('service-page', array(
    'rewrite' => array('slug' => 'services'),
    'has_archive' => true,
    // ... other args
));
```

### Custom Archive Templates
Create specific archive templates for each subsection:
- `archive-about-page.php`
- `archive-service-page.php`

### Custom Settings Pages
Each subsection can have its own settings page with specific options.

## Benefits

1. **SEO Friendly**: Clean URLs like `/subsection` instead of `/page/subsection`
2. **GenerateBlocks Compatible**: Works seamlessly with location rules
3. **Easy Management**: Simple settings page for homepage selection
4. **Flexible Content**: Full WordPress editor with blocks support
5. **Admin Integration**: Edit link in admin bar points to correct page
6. **Scalable**: Can create multiple subsections easily

## Example Implementation

Here's how this pattern was implemented in a real project:
- Post Type: `subsection-page` with slug `subsection`
- Settings: `inc/subsection-settings.php`
- Template: `archive-subsection-page.php`
- URL: `/subsection` displays selected homepage content

## Troubleshooting

### Permalinks Not Working
Run: `wp rewrite flush` or visit Settings > Permalinks and save

### GenerateBlocks Not Applying
Ensure location rules are set to "Post Type Archive: [Your Post Type]"

### Content Not Displaying
Check that a homepage is selected in the subsection settings

### Admin Bar Edit Link Not Working
Verify the `wp_admin_bar_edit_menu_args` filter is in the archive template 