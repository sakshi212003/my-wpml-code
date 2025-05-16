<?php
/* Template Name: Submit Accident News */

if (post_password_required()) {
    get_header();
    echo get_the_password_form();
    get_footer();
    exit;
}

get_header();
wp_enqueue_media();
session_start();

$form_submitted = false;

// Get current WPML language
if (defined('ICL_LANGUAGE_CODE')) {
    $language = ICL_LANGUAGE_CODE;
} else {
    $language = apply_filters('wpml_current_language', NULL);
}

if ($_SERVER["REQUEST_METHOD"] == "POST") {
    $title = isset($_POST['title']) ? sanitize_text_field($_POST['title']) : '';
    $type_of_accident = isset($_POST['type_of_accident']) ? wp_kses_post($_POST['type_of_accident']) : '';
    $article_summary = isset($_POST['article_summary']) ? wp_kses_post($_POST['article_summary']) : '';
    $news_source_link = isset($_POST['news_source_link']) ? esc_url($_POST['news_source_link']) : '';
    $featured_category = isset($_POST['featured_image_category']) ? sanitize_text_field($_POST['featured_image_category']) : 'default';

    $default_image_url = 'https://example.com/wp-content/upwwwwwloads/202d3335/02/police-car-sirens-768x481.jpg';

    $category_folders = [
        'car_accident'        => 'car-accidents',
        'motorcycle_accident' => 'motorcycle-accidents',
        'pedestrian_accident' => 'pedestrian-accidents',
        'truck_accident'      => 'truck-accidents'
    ];

    function get_random_featured_image_url($subfolder) {
        $upload_dir = wp_upload_dir();
        $folder_path = $upload_dir['basedir'] . '/accident-images/' . $subfolder;
        $folder_url  = $upload_dir['baseurl'] . '/accident-images/' . $subfolder;

        if (!is_dir($folder_path)) return null;

        $images = glob($folder_path . '/*.{jpg,jpeg,png,gif}', GLOB_BRACE);
        if (!$images || empty($images)) return null;

        $random_image = $images[array_rand($images)];
        return $folder_url . '/' . basename($random_image);
    }

    $featured_image_url = ($featured_category === 'default' || !isset($category_folders[$featured_category]))
        ? $default_image_url
        : get_random_featured_image_url($category_folders[$featured_category]) ?? $default_image_url;

    $shawn_user_id = 1;
    $current_user_id = is_user_logged_in() ? get_current_user_id() : $shawn_user_id;

    $post_data = [
        'post_title'    => $title,
        'post_content'  => '',
        'post_status'   => 'publish',
        'post_category' => [get_cat_ID('Accident News')],
        'post_type'     => 'post',
        'post_author'   => $current_user_id,
    ];

    $post_id = wp_insert_post($post_data);

    if ($post_id) {
        // Assign WPML language to the post
        do_action('wpml_set_element_language_details', [
            'element_id'           => $post_id,
            'element_type'         => 'post_' . get_post_type($post_id),
            'trid'                 => null,
            'language_code'        => $language,
            'source_language_code' => null,
        ]);

        // Include media dependencies
        require_once(ABSPATH . 'wp-admin/includes/image.php');
        require_once(ABSPATH . 'wp-admin/includes/file.php');
        require_once(ABSPATH . 'wp-admin/includes/media.php');

        $image_html = '';
        $media_id = null;

        $tmp = download_url($featured_image_url);

        if (!is_wp_error($tmp)) {
            $file_array = [
                'name'     => basename($featured_image_url),
                'tmp_name' => $tmp
            ];

            $media_id = media_handle_sideload($file_array, $post_id);
            if (!is_wp_error($media_id)) {
                set_post_thumbnail($post_id, $media_id);

                $src = wp_get_attachment_image_url($media_id, 'full');
                $srcset = wp_get_attachment_image_srcset($media_id, 'full');
                $sizes = '(max-width: 480px) 100vw, (max-width: 768px) 80vw, 1200px';

                $image_html .= "<div class='community-featured-container'>";
                $image_html .= "<img class='community-featured-logo' src='" . esc_url($src) . "' srcset='" . esc_attr($srcset) . "' sizes='" . esc_attr($sizes) . "' alt='Accident Image'>";
                $image_html .= "<p class='community-featured-caption'>This is not a depiction of this particular incident.</p>";
                $image_html .= "</div>";
            }
        }

        if (!$image_html) {
            $image_html .= "<div class='community-featured-container'>";
            $image_html .= "<img class='community-featured-logo' src='" . esc_url($featured_image_url) . "' alt='Accident Image'>";
            $image_html .= "<p class='community-featured-caption'>This is not a depiction of this particular incident.</p>";
            $image_html .= "</div>";
        }

        $content = $image_html;
        $content .= "<h4><strong>Type of Accident:</strong></h4>" . wpautop($type_of_accident);
        $content .= "<h4><strong>Article Summary:</strong></h4>" . wpautop($article_summary);
        $content .= "<p><em>(<a href='" . esc_url($news_source_link) . "' target='_blank' rel='noopener'>Source</a>)</em></p>";

        wp_update_post([
            'ID'           => $post_id,
            'post_content' => $content,
        ]);

        // 🔄 WPML automatic translation to English <-> Spanish
        if (defined('WPML_ST_VERSION')) {
            $translation_service = apply_filters('wpml_get_auto_translate_service', null);
            $languages = ['en', 'es'];

            foreach ($languages as $target_lang) {
                if ($target_lang !== $language && $translation_service) {
                    $translation_service->auto_translate($post_id, $target_lang);
                }
            }
        }

        if (function_exists('do_action')) {
            do_action('breeze_clear_all_cache');
        }

        $_SESSION['form_submitted'] = true;
        wp_redirect(add_query_arg('success', 'true', get_permalink()));
        exit;
    }
}

?>
