import 'package:flutter/material.dart';
import 'dart:math';
import 'dart:convert';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:intl/intl.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:google_sign_in/google_sign_in.dart';
import 'package:flutter_colorpicker/flutter_colorpicker.dart';
import 'package:url_launcher/url_launcher.dart'; // For launchUrl
import 'package:flutter/services.dart'; // For Clipboard
import 'package:flutter/cupertino.dart';
// import 'package:confetti/confetti.dart';
import 'dart:ui'; // <--- ADD THIS IMPORT for PathMetric
import 'package:flutter/widgets.dart';
import 'package:collection/collection.dart';
import 'dart:async';

const int MAX_CONFIDENCE_SCORE = 880; // Global Constant

final List<MilestoneDefinition> GAME_MILESTONES = [
  // INDEX 0: FIRST MILESTONE - REQUIRES 40 POINTS (Locked at start!)
  MilestoneDefinition(
    id: 'awakening',
    titleKey: 'the_awakening',
    rewardKey: 'seed_badge',
    icon: Icons.psychology_alt,
    requiredScore: 40,
    bonusPoints: 10,
    color: Colors.brown.shade400,
  ),

  // INDEX 1: 140 Points
  MilestoneDefinition(
    id: 'first_spark',
    titleKey: 'first_spark',
    rewardKey: 'bronze_leaf',
    icon: Icons.flash_on,
    requiredScore: 140,
    bonusPoints: 25,
    color: Colors.deepOrange,
  ),

  // INDEX 2: 340 Points
  MilestoneDefinition(
    id: 'social_courage',
    titleKey: 'social_courage',
    rewardKey: 'silver_sprout',
    icon: Icons.shield,
    requiredScore: 340,
    bonusPoints: 50,
    color: Colors.grey.shade400,
  ),

  // INDEX 3: 580 Points
  MilestoneDefinition(
    id: 'confidence_bloom',
    titleKey: 'confidence_bloom',
    rewardKey: 'gold_flower',
    icon: Icons.local_florist,
    requiredScore: 580,
    bonusPoints: 100,
    color: Colors.amber,
  ),

  // INDEX 4: 880 Points (MAX)
  MilestoneDefinition(
    id: 'mastery',
    titleKey: 'mastery',
    rewardKey: 'diamond_crown',
    icon: Icons.diamond,
    requiredScore: 880,
    bonusPoints: 250,
    color: Colors.cyanAccent.shade400,
  ),
];

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  await GlobalSettings.load();
  runApp(const BloomApp());
}

// --- GLOBAL STATE ---
// REPLACE THE ENTIRE GlobalSettings CLASS WITH THIS:
class GlobalSettings {
  // --- NOTIFIERS ---
  static ValueNotifier<Color> themeColor = ValueNotifier(
    const Color(0xFF00897B),
  );
  static ValueNotifier<String> language = ValueNotifier('en');
  static ValueNotifier<ThemeMode> themeMode = ValueNotifier(ThemeMode.light);
  static ValueNotifier<Map<String, dynamic>> userProfile = ValueNotifier({});

  // --- LOCAL STORAGE INSTANCE ---
  static final LocalStorageService _storage = LocalStorageService();

  static const List<Color> themePalette = [
    Color(0xFF00897B),
    Color(0xFF673AB7),
    Color(0xFFF57C00),
    Color(0xFF388E3C),
    Color(0xFF1976D2),
    Color(0xFFE91E63),
    Color(0xFF455A64),
    Color.fromARGB(255, 241, 219, 91),
    Color.fromARGB(255, 192, 137, 255),
    Color.fromARGB(255, 1, 58, 85),
    Color.fromARGB(255, 207, 14, 14),
  ];

  // --- LOAD METHOD (USES SHAREDPREFERENCES DIRECTLY LIKE YOUR ORIGINAL) ---
  static Future<void> load() async {
    await _storage.init();
    SharedPreferences prefs = await SharedPreferences.getInstance();
    themeColor.value = Color(
      prefs.getInt('themeColor') ?? const Color(0xFF00897B).value,
    );
    themeMode.value = ThemeMode.values[prefs.getInt('themeMode') ?? 0];
    language.value = prefs.getString('language') ?? 'en';

    var localProfile = _storage.loadProfile();
    if (localProfile.isNotEmpty) {
      // Ensure new fields exist
      localProfile['claimedMilestones'] ??= <String>[];
      localProfile['levelTaskCounts'] ??= <String, int>{};
      userProfile.value = localProfile;
    } else {
      userProfile.value = {'levelTaskCounts': <String, int>{}};
    }
  }

  // --- SAVE METHODS (USE SHAREDPREFERENCES DIRECTLY) ---
  static Future<void> saveTheme(Color c) async {
    SharedPreferences prefs = await SharedPreferences.getInstance();
    await prefs.setInt('themeColor', c.value);
    themeColor.value = c;
  }

  static Future<void> saveLanguage(String l) async {
    SharedPreferences prefs = await SharedPreferences.getInstance();
    await prefs.setString('language', l);
    language.value = l;
  }

  static Future<void> saveThemeMode(ThemeMode m) async {
    SharedPreferences prefs = await SharedPreferences.getInstance();
    await prefs.setInt('themeMode', m.index);
    themeMode.value = m;
  }

  // --- PROFILE HELPERS ---
  static Future<void> updateName(String name) async {
    var profile = Map<String, dynamic>.from(userProfile.value);
    profile['userName'] = name;
    await _storage.saveProfile(profile);
    userProfile.value = profile;
  }

  static Future<void> completeTaskLocal(String taskId, double points) async {
    var profile = Map<String, dynamic>.from(userProfile.value);
    profile = StreakEngine.onTaskCompleted(profile, taskId, points);
    await _storage.saveProfile(profile);
    userProfile.value = profile;
  }

  // NEW: Call this when user claims a milestone reward
  static Future<void> claimMilestoneLocal(
    String milestoneId,
    int bonusPoints,
  ) async {
    var profile = Map<String, dynamic>.from(userProfile.value);

    // 1. Add Points
    profile = StreakEngine.addBonusPoints(profile, bonusPoints);

    // 2. Mark as claimed
    List<String> claimed = List<String>.from(
      profile['claimedMilestones'] ?? [],
    );
    if (!claimed.contains(milestoneId)) {
      claimed.add(milestoneId);
    }
    profile['claimedMilestones'] = claimed;

    await _storage.saveProfile(profile);
    userProfile.value = profile;
  }

  static Future<void> resetLocalSettings() async {
    await _storage.clearLocalProfile();
    themeColor.value = const Color(0xFF00897B);
    language.value = 'en';
    themeMode.value = ThemeMode.light;
    userProfile.value = {};
  }
}

// --- ANIMATION WRAPPER ---
// class FadeScale extends StatefulWidget {
//   final Widget child;
//   const FadeScale({super.key, required this.child});

//   @override
//   State<FadeScale> createState() => _FadeScaleState();
// }

// class _FadeScaleState extends State<FadeScale> {
//   double _opacity = 0.0;
//   double _scale = 0.95;

//   @override
//   void initState() {
//     super.initState();
//     Future.delayed(const Duration(milliseconds: 50), () {
//       if (mounted)
//         setState(() {
//           _opacity = 1.0;
//           _scale = 1.0;
//         });
//     });
//   }

//   @override
//   Widget build(BuildContext context) {
//     return AnimatedOpacity(
//       duration: const Duration(milliseconds: 600),
//       opacity: _opacity,
//       child: AnimatedScale(
//         duration: const Duration(milliseconds: 600),
//         scale: _scale,
//         child: widget.child,
//       ),
//     );
//   }
// }

class SplashScreen extends StatefulWidget {
  const SplashScreen({super.key});

  @override
  State<SplashScreen> createState() => _SplashScreenState();
}

class _SplashScreenState extends State<SplashScreen>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;

  // Define animation intervals for a staggered sequence
  late final Animation<double> _logoScaleAnim;
  late final Animation<double> _logoFadeAnim;
  late final Animation<Offset> _textSlideAnim;
  late final Animation<double> _textFadeAnim;
  late final Animation<double> _progressFadeAnim;

  @override
  void initState() {
    super.initState();

    _controller = AnimationController(
      duration: const Duration(milliseconds: 2200), // Total sequence duration
      vsync: this,
    );

    // Adjust these intervals proportionally (2200/1600 = 1.375x)
    // 1. Logo: Scale up with elastic curve (0.0s - 0.6s)
    _logoScaleAnim = CurvedAnimation(
      parent: _controller,
      curve: const Interval(0.0, 0.8, curve: Curves.elasticOut),
    );
    // Logo fade in slightly faster
    _logoFadeAnim = CurvedAnimation(
      parent: _controller,
      curve: const Interval(0.0, 0.6, curve: Curves.easeOut),
    );

    // 2. Text: Slide up + Fade in (0.4s - 1.0s)
    _textSlideAnim =
        Tween<Offset>(begin: const Offset(0, 0.7), end: Offset.zero).animate(
          CurvedAnimation(
            parent: _controller,
            curve: const Interval(0.4, 0.9, curve: Curves.easeOutCubic),
          ),
        );
    _textFadeAnim = CurvedAnimation(
      parent: _controller,
      curve: const Interval(0.4, 0.9, curve: Curves.easeOut),
    );

    // 3. Progress Bar: Fade in last (0.8s - 1.0s)
    _progressFadeAnim = CurvedAnimation(
      parent: _controller,
      curve: const Interval(0.6, 1.0, curve: Curves.easeIn),
    );

    _controller.forward();
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    // Listen to theme/language for dynamic updates
    return ValueListenableBuilder<Color>(
      valueListenable: GlobalSettings.themeColor,
      builder: (context, themeColor, _) {
        return ValueListenableBuilder<String>(
          valueListenable: GlobalSettings.language,
          builder: (context, lang, _) {
            // Use Theme's surface color for solid background (fixes transition flash)
            final surfaceColor = Theme.of(context).colorScheme.surface;
            final welcomeText = AppTexts.get('welcome', lang);
            final subtitleText = AppTexts.get('subtitle', lang);

            return Scaffold(
              backgroundColor:
                  surfaceColor, // Solid background prevents transition flash
              body: Center(
                child: Column(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    // 1. LOGO: Scale + Fade
                    ScaleTransition(
                      scale: _logoScaleAnim,
                      child: FadeTransition(
                        opacity: _logoFadeAnim,
                        child: Icon(
                          Icons.wb_sunny_rounded,
                          size: 100,
                          color: themeColor, // Use global theme color
                        ),
                      ),
                    ),
                    const SizedBox(height: 24),

                    // 2. TEXT: Slide Up + Fade
                    SlideTransition(
                      position: _textSlideAnim,
                      child: FadeTransition(
                        opacity: _textFadeAnim,
                        child: Column(
                          children: [
                            Text(
                              AppTexts.get('welcome', lang),
                              style: const TextStyle(
                                fontSize: 32,
                                fontWeight: FontWeight.bold,
                                color: Colors
                                    .white, // Splash uses themeColor background, so white text
                                fontFamily: 'Georgia',
                              ),
                              textAlign: TextAlign.center,
                            ),
                            const SizedBox(height: 12),
                            Padding(
                              padding: const EdgeInsets.symmetric(
                                horizontal: 40,
                              ),
                              child: Text(
                                AppTexts.get('subtitle', lang),
                                style: TextStyle(
                                  fontSize: 16,
                                  color: Colors.white.withOpacity(0.9),
                                  fontFamily: 'Georgia',
                                ),
                                textAlign: TextAlign.center,
                              ),
                            ),
                          ],
                        ),
                      ),
                    ),

                    const SizedBox(height: 48),

                    // 3. PROGRESS BAR: Fade In
                    FadeTransition(
                      opacity: _progressFadeAnim,
                      child: SizedBox(
                        width: 200,
                        child: Column(
                          children: [
                            LinearProgressIndicator(
                              valueColor: AlwaysStoppedAnimation<Color>(
                                Colors.white,
                              ),
                              backgroundColor: Colors.white.withOpacity(0.2),
                              minHeight: 4,
                              borderRadius: BorderRadius.circular(2),
                            ),
                            const SizedBox(height: 8),
                            Text(
                              AppTexts.get(
                                'loading',
                                GlobalSettings.language.value,
                              ), // Add 'loading' key to AppTexts
                              style: TextStyle(
                                fontSize: 12,
                                color: Colors.white.withOpacity(0.8),
                                fontFamily: 'Georgia',
                              ),
                            ),
                          ],
                        ),
                      ),
                    ),
                  ],
                ),
              ),
            );
          },
        );
      },
    );
  }
}

class LocalStorageService {
  static const String _kProfileKey = 'bloom_user_profile';
  late SharedPreferences _prefs;

  Future<void> init() async {
    _prefs = await SharedPreferences.getInstance();
  }

  // Saves the entire profile map to the device
  Future<void> saveProfile(Map<String, dynamic> profile) async {
    if (_prefs == null) await init(); // <--- AUTO-INIT FIX
    await _prefs.setString(_kProfileKey, jsonEncode(profile));
  }

  // Save the UID of the last guest user
  Future<void> saveGuestUid(String uid) async {
    await _prefs.setString('last_guest_uid', uid);
  }

  // Retrieve the UID of the last guest user
  String? getGuestUid() {
    return _prefs.getString('last_guest_uid');
  }

  Future<void> clearLocalProfile() async {
    await _prefs.remove(_kProfileKey);
  }

  // Loads the profile from the device
  Map<String, dynamic> loadProfile() {
    // Safety: Ensure prefs is initialized (should be ready after GlobalSettings.load())
    if (_prefs == null) {
      // Fallback synchronous init (rare race condition)
      _prefs =
          SharedPreferences.getInstance()
              as dynamic; // Hack for sync, or just return empty
      // Better: Just return empty if somehow not ready, GlobalSettings.load() awaits init().
      return {};
    }
    final str = _prefs.getString(_kProfileKey);
    if (str == null) return {};
    try {
      return Map<String, dynamic>.from(jsonDecode(str));
    } catch (_) {
      return {};
    }
  }
}

// --- STREAK ENGINE (Add this new class) ---
// --- STREAK ENGINE (REPLACE ENTIRE CLASS) ---
class StreakEngine {
  static const double _decayPenalty = 2.0;

  /// Called on App Start/Resume -> Handles DECAY/FREEZE only
  static Map<String, dynamic> processDailyUpdate(Map<String, dynamic> profile) {
    final todayStr = DateFormat('yyyy-MM-dd').format(DateTime.now());
    final lastDateStr = profile['lastCompletedDate'] as String? ?? '';

    if (lastDateStr.compareTo(todayStr) >= 0) return profile;

    final lastDate =
        DateTime.tryParse(lastDateStr) ??
        DateTime.now().subtract(const Duration(days: 100));
    final diff = DateTime.now().difference(lastDate).inDays;

    if (diff > 1) {
      int activeFreezes = profile['activeFreezes'] as int? ?? 0;
      int currentStreak = profile['currentStreak'] as int? ?? 0;
      double score = (profile['confidenceScore'] as num?)?.toDouble() ?? 0.0;

      if (activeFreezes > 0) {
        activeFreezes -= 1;
      } else {
        currentStreak = 0;
        score = max(0.0, score - _decayPenalty);
      }

      return {
        ...profile,
        'currentStreak': currentStreak,
        'activeFreezes': activeFreezes,
        'confidenceScore': score,
      };
    }
    return profile;
  }

  /// Called when Task Completed -> Handles STREAK INCREMENT
  static Map<String, dynamic> onTaskCompleted(
    Map<String, dynamic> profile,
    String taskId,
    double points,
  ) {
    List<String> completed = List<String>.from(profile['completedTasks'] ?? []);
    if (completed.contains(taskId)) return profile;

    completed.add(taskId);
    double newScore =
        ((profile['confidenceScore'] as num?)?.toDouble() ?? 0.0) + points;
    int newPoints =
        ((profile['totalPoints'] as num?)?.toInt() ?? 0) + points.round();

    final todayStr = DateFormat('yyyy-MM-dd').format(DateTime.now());
    final lastDateStr = profile['lastCompletedDate'] as String? ?? '';

    int currentStreak = profile['currentStreak'] as int? ?? 0;
    int bestStreak = profile['bestStreak'] as int? ?? 0;

    if (lastDateStr == todayStr) {
      // Same day, streak unchanged
    } else if (lastDateStr ==
        DateFormat(
          'yyyy-MM-dd',
        ).format(DateTime.now().subtract(const Duration(days: 1)))) {
      currentStreak += 1;
      bestStreak = max(bestStreak, currentStreak);
    } else {
      currentStreak = 1;
      bestStreak = max(bestStreak, currentStreak);
    }

    return {
      ...profile,
      'confidenceScore': newScore,
      'totalPoints': newPoints,
      'completedTasks': completed,
      'lastCompletedDate': todayStr,
      'currentStreak': currentStreak,
      'bestStreak': bestStreak,
    };
  }

  /// NEW: Adds bonus points (for milestones) without affecting streak
  static Map<String, dynamic> addBonusPoints(
    Map<String, dynamic> profile,
    int bonus,
  ) {
    double newScore =
        ((profile['confidenceScore'] as num?)?.toDouble() ?? 0.0) + bonus;
    int newPoints = ((profile['totalPoints'] as num?)?.toInt() ?? 0) + bonus;
    return {...profile, 'confidenceScore': newScore, 'totalPoints': newPoints};
  }

  static Map<String, dynamic> buyFreeze(
    Map<String, dynamic> profile,
    int cost,
  ) {
    int points = profile['totalPoints'] as int? ?? 0;
    if (points < cost) return profile;
    return {
      ...profile,
      'totalPoints': points - cost,
      'streakFreezes': (profile['streakFreezes'] as int? ?? 0) + 1,
    };
  }

  static Map<String, dynamic> equipFreeze(Map<String, dynamic> profile) {
    int owned = profile['streakFreezes'] as int? ?? 0;
    int active = profile['activeFreezes'] as int? ?? 0;
    if (owned <= 0 || active >= 2) return profile;
    return {
      ...profile,
      'streakFreezes': owned - 1,
      'activeFreezes': active + 1,
    };
  }
}

// Add this class near the top of your file (e.g., after FadeScale)
class Tr extends StatelessWidget {
  final String keyText;
  final TextStyle? style;
  final TextAlign? textAlign; // ADDED
  final int? maxLines; // ADDED
  final TextOverflow? overflow; // ADDED
  final bool? softWrap; // ADDED
  final List<String>? args; // For interpolation

  const Tr(
    this.keyText, {
    super.key,
    this.style,
    this.textAlign, // ADDED
    this.maxLines, // ADDED
    this.overflow, // ADDED
    this.softWrap, // ADDED
    this.args,
  });

  @override
  Widget build(BuildContext context) {
    return ValueListenableBuilder<String>(
      valueListenable: GlobalSettings.language,
      builder: (context, lang, _) {
        String text = AppTexts.get(keyText, lang);
        if (args != null) {
          for (int i = 0; i < args!.length; i++) {
            text = text.replaceAll('{$i}', args![i]);
          }
        }
        return Text(
          text,
          style: style,
          textAlign: textAlign, // FORWARDED
          maxLines: maxLines, // FORWARDED
          overflow: overflow, // FORWARDED
          softWrap: softWrap, // FORWARDED
        );
      },
    );
  }
}

// --- TRANSLATIONS ---
class AppTexts {
  static final Map<String, Map<String, String>> translations = {
    'en': {
      // Progress Screen
      'your_journey': 'Your Journey',
      'keep_growing_sub': 'Every small step is a victory. Keep growing!',
      'how_it_works': 'How does it work?',
      'total_points': 'Total Points',
      'current_streak': 'Current Streak',
      'tasks_done': 'Tasks Done',
      'rank': 'Rank',
      'days': 'Days',
      'contact_us': 'Contact Us',
      'contact_email_prompt': 'For support and feedback, email us at:',
      'close': 'Close',
      // Level Map Screen
      'tap_to_view_journey': 'Tap to view your journey! 🌸',
      'tap_to_start': 'Tap to start challenge',
      'choose_level': 'Choose your growth stage:',
      'view_journey': 'View Growth Journey',
      'progress': 'Your Growth Progress',
      'level_seedling': 'Seedling',
      'level_sprout': 'Sprout',
      'level_leaf': 'Leaf',
      'level_stem': 'Stem',
      'level_bloom': 'Bloom',
      // Profile Screen
      'account': 'Account',
      'display_name': 'Display Name',
      'save_name': 'Save Name',
      'app_theme': 'App Theme',
      'select_color': 'Select your Bloom Color:',
      'light_mode': 'Light',
      'dark_mode': 'Dark',
      'language': 'Language',
      'logout': 'Logout',
      'profile_updated': 'Profile updated!',
      'pick_theme_color': 'Pick a theme color',
      'done': 'Done',
      'reset_all_progress': 'Reset All Progress',
      'reset_confirm_title': 'Are you sure?',
      'reset_confirm_message':
          'This will permanently delete your confidence score, streak, and all history. This cannot be undone.',
      'cancel': 'Cancel',
      'reset_everything': 'Reset Everything',
      'reset_success': 'All progress has been reset.',
      // Shop Screen
      'shop_title': 'Bloom Shop',
      'your_points': 'Your Points',
      'available_items': 'Available Items',
      'streak_freeze': 'Streak Freeze',
      'protects_streak': 'Protects streak from resetting',
      'your_inventory': 'Your Inventory',
      'owned': 'Owned',
      'equipped': 'Equipped',
      'equip_freeze': 'Equip Freeze',
      'buy': 'Buy',
      'freeze_purchased': 'Freeze Purchased!',
      'not_enough_points': 'Not enough points!',
      'freeze_equipped': 'Freeze equipped!',
      // Milestone Screen
      'your_growth_path': 'Your Growth Path',
      'the_awakening': 'The Awakening',
      'seed_badge': 'Seed Badge',
      'first_spark': 'First Spark',
      'bronze_leaf': 'Bronze Leaf',
      'social_courage': 'Social Courage',
      'silver_sprout': 'Silver Sprout',
      'confidence_bloom': 'Confidence Bloom',
      'gold_flower': 'Gold Flower',
      'mastery': 'Mastery',
      'diamond_crown': 'Diamond Crown',
      'milestone_claimed': 'Claimed',
      'milestone_need_score': 'Need {0}pts',
      'milestone_claim_reward': 'Claim +{0} pts',
      'milestone_reward_toast': 'Claimed! +{0} pts',
      // FAQ Screen
      'help_faq': 'Help and FAQs',
      'common_questions': 'Common Questions',
      'keep_blooming': 'Keep Blooming! 🌸',
      'faq_q1': 'What is Bloom?',
      'faq_a1':
          'Bloom is a self-help tool designed to help people reduce social anxiety through a process called \'Graded Exposure.\' By completing small, manageable social tasks, you train your brain to realize that social interactions are safe and manageable.',
      'faq_q2': 'How do the levels work?',
      'faq_a2':
          'We start with \'Seedling\' (very easy tasks) and move up to \'Bloom\' (more challenging tasks). As you complete tasks, you earn confidence points. The higher the level, the more points you earn!',
      'faq_q3': 'What is the Confidence Meter?',
      'faq_a3':
          'The progress bar on your home screen represents your overall confidence. It grows as you complete tasks. Be careful: if you stop practicing for several days, your confidence score may decay slightly, reminding you that confidence is a muscle that needs regular exercise!',
      'faq_q4': 'What is a Streak?',
      'faq_a4':
          'A streak is a count of how many consecutive days you have completed at least one task. Consistency is the key to overcoming anxiety, so try to keep your flame burning!',
      'faq_q5': 'Where is my data stored?',
      'faq_a5':
          'Your privacy is our priority. All your progress, history, and profile data are stored locally on your own device. Nothing is uploaded to a cloud server.',
      'faq_q6': 'What do I do if the app crashes?',
      'faq_a6':
          'If the app behaves strangely, try restarting your phone. If you\'ve updated the app, you might need to clear the app cache in your Android settings. If all else fails, you can use the \'Reset All Progress\' option in your Profile.',
      'faq_q7': 'Can I skip levels?',
      'faq_a7':
          'Yes! While we recommend the gradual path, you are free to choose any level from the map that feels appropriate for your current comfort level.',
      'faq_q8': 'What if the task is very hard?',
      'faq_a8':
          'While we recommend you to try to accomplish the task, you can just go back to home screen, and re-enter to change the current task',
      'faq_q9': 'Contact Us',
      'faq_a9':
          'We would love to hear about our app from our users, recommendations for the future updates. We\'d love to hear how good the app works for the users, and what it lacks and require improvement. Please feel free to share feedback on feedback.bloom@gmail.com, we\'d really appreciate it',
      // Task Screen
      'stage_label': 'Stage: {0}',
      'keep_growing': 'Keep growing, {0}',
      'current_challenge': 'Your current challenge for this stage:',
      'stage_mastered': 'Stage Mastered!',
      'all_done': 'You\'ve completed all challenges in this stage.',
      'return_map': 'Return to Map',
      'well_done': 'Well Done!',
      'i_completed': 'I Completed This',
      'level_up_suggestion_title': 'Level Up Suggestion',
      'level_up_suggestion_message':
          'You\'ve completed 10 tasks in this level! You\'re ready for the next level. Want to move up?',
      'stay_here': 'Stay Here',
      'move_to_next_level': 'Move to Next Level',
      // Reflection Screen
      'reflect': 'Reflect on your growth',
      'challenge': 'Challenge',
      'anxiety_q': 'How anxious did you feel? (1-10)',
      'what_happened': 'What actually happened?',
      'write_experience_hint': 'Write about your experience...',
      'finish': 'Finish Reflection',
      // General / Auth
      'welcome': 'Welcome to Bloom',
      'subtitle': 'A safe space to grow your confidence.',
      'start': 'Start My Journey',
      'guest': 'Continue as Guest',
      'hello': 'Hello',
      'profile': 'My Profile',
      'history': 'My Growth Journey',
      'streak': 'Current Streak',
      'best': 'Best Streak',
      'points': 'Confidence Points',
      //Splash Screen
      'loading': 'Loading your garden...',
    },
    'es': {
      // Progress Screen
      'your_journey': 'Tu viaje',
      'keep_growing_sub':
          'Cada pequeño paso es una victoria. ¡Sigue creciendo!',
      'how_it_works': '¿Cómo funciona?',
      'total_points': 'Puntos totales',
      'current_streak': 'Racha actual',
      'tasks_done': 'Tareas realizadas',
      'rank': 'Rango',
      'days': 'Días',
      'contact_us': 'Contacta con nosotros',
      'contact_email_prompt':
          'Para obtener asistencia y enviar comentarios, escríbanos a:',
      'close': 'Cerca',
      // Level Map Screen
      'tap_to_view_journey': '¡Toca para ver tu viaje! 🌸',
      'tap_to_start': 'Toca para empezar el desafío',
      'choose_level': 'Elige tu etapa de crecimiento:',
      'view_journey': 'Ver viaje de crecimiento',
      'progress': 'Tu progreso de crecimiento',
      'level_seedling': 'Brotación',
      'level_sprout': 'Brote',
      'level_leaf': 'Hoja',
      'level_stem': 'Tallo',
      'level_bloom': 'Floración',
      // Profile Screen
      'account': 'Cuenta',
      'display_name': 'Nombre de pantalla',
      'save_name': 'Guardar nombre',
      'app_theme': 'Tema de la aplicación',
      'select_color': 'Selecciona tu color Bloom:',
      'light_mode': 'Claro',
      'dark_mode': 'Oscuro',
      'language': 'Idioma',
      'logout': 'Cerrar sesión',
      'profile_updated': 'Perfil actualizado!',
      'pick_theme_color': 'Elige un color de tema',
      'done': 'Hecho',
      'reset_all_progress': 'Restablecer todo el progreso',
      'reset_confirm_title': 'Estas seguro',
      'reset_confirm_message':
          'Esto eliminará permanentemente tu puntuación de confianza, tu racha y todo tu historial. Esta acción es irreversible.',
      'cancel': 'Cancelar',
      'reset_everything': 'Restablecer todo',
      'reset_success': 'Se ha reiniciado todo el progreso.',
      // Shop Screen
      'shop_title': 'Bloom Shop',
      'your_points': 'Tus puntos',
      'available_items': 'Artículos disponibles',
      'streak_freeze': 'Protector de racha',
      'protects_streak': 'Evita que la racha se reinicie',
      'your_inventory': 'Tu inventario',
      'owned': 'Comprado',
      'equipped': 'Equipado',
      'equip_freeze': 'Equipar protector',
      'buy': 'Comprar',
      'freeze_purchased': 'Congelar comprado!',
      'not_enough_points': 'No hay suficientes puntos!',
      'freeze_equipped': 'Congelar equipado!',
      // Milestone Screen
      'your_growth_path': 'Tu camino de crecimiento',
      'the_awakening': 'El despertar',
      'seed_badge': 'Insignia de semilla',
      'first_spark': 'Primera chispa',
      'bronze_leaf': 'Hoja de bronce',
      'social_courage': 'Coraje social',
      'silver_sprout': 'Brote de plata',
      'confidence_bloom': 'Flor de confianza',
      'gold_flower': 'Flor de oro',
      'mastery': 'Maestría',
      'diamond_crown': 'Corona de diamante',
      'milestone_claimed': 'Reclamado',
      'milestone_need_score': 'Se necesita {0}%',
      'milestone_claim_reward': 'Reclamar +{0} pts',
      'milestone_reward_toast': '¡Reclamado! +{0} pts',
      // FAQ Screen
      'help_faq': 'Ayuda y preguntas frecuentes',
      'common_questions': 'Preguntas frecuentes',
      'keep_blooming': '¡Sigue floreciendo! 🌸',
      'faq_q1': '¿Qué es la floración?',
      'faq_a1':
          'Bloom es una herramienta de autoayuda diseñada para ayudar a las personas a reducir la ansiedad social a través de un proceso llamado \'Exposición Gradual\'. Al completar tareas sociales pequeñas y manejables, entrenas a tu cerebro para que se dé cuenta de que las interacciones sociales son seguras y manejables.',
      'faq_q2': '¿Cómo funcionan los niveles?',
      'faq_a2':
          'Comenzamos con \'Plántula\' (tareas muy fáciles) y avanzamos hasta \'Floración\' (tareas más desafiantes). A medida que completas tareas, ganas puntos de confianza. ¡Cuanto más alto sea el nivel, más puntos ganarás!',
      'faq_q3': '¿Qué es el Medidor de Confianza?',
      'faq_a3':
          'La barra de progreso en tu pantalla de inicio representa tu confianza general. Crece a medida que completas tareas. Ten cuidado: si dejas de practicar durante varios días, tu puntuación de confianza puede disminuir ligeramente, ¡recordándote que la confianza es un músculo que necesita ejercicio regular!',
      'faq_q4': '¿Qué es una Racha?',
      'faq_a4':
          'Una racha es el recuento de cuántos días consecutivos has completado al menos una tarea. La constancia es la clave para superar la ansiedad, ¡así que intenta mantener tu llama encendida!',
      'faq_q5': '¿Dónde se guardan mis datos?',
      'faq_a5':
          'Tu privacidad es nuestra prioridad. Todo tu progreso, historial y datos de perfil se guardan localmente en tu propio dispositivo. Nada se sube a un servidor en la nube.',
      'faq_q6': '¿Qué hago si la aplicación se cierra inesperadamente?',
      'faq_a6':
          'Si la aplicación se comporta de forma extraña, intenta reiniciar tu teléfono. Si has actualizado la aplicación, es posible que debas borrar la caché de la aplicación en los ajustes de tu Android. Si todo lo demás falla, puedes usar la opción \'Restablecer todo el progreso\' en tu Perfil.',
      'faq_q7': '¿Puedo saltarme niveles?',
      'faq_a7':
          '¡Sí! Aunque recomendamos el camino gradual, eres libre de elegir cualquier nivel del mapa que se adapte a tu nivel de comodidad actual.',
      'faq_q8': '¿Qué pasa si la tarea es muy difícil?',
      'faq_a8':
          'Aunque te recomendamos que intentes realizar la tarea, puedes simplemente volver a la pantalla de inicio y volver a entrar para cambiar la tarea actual.',
      'faq_q9': 'Contáctanos',
      'faq_a9':
          'Nos encantaría saber qué piensan nuestros usuarios sobre la aplicación y recibir recomendaciones para futuras actualizaciones. Nos gustaría saber qué tan bien les funciona la aplicación a los usuarios, y qué le falta o necesita mejorar. No dudes en compartir tus comentarios en feedback.bloom@gmail.com, realmente lo agradeceríamos.',
      // Task Screen
      'stage_label': 'Etapa: {0}',
      'keep_growing': 'Sigue creciendo, {0}',
      'current_challenge': 'Tu desafío actual para esta etapa:',
      'stage_mastered': '¡Etapa dominada!',
      'all_done': 'Has completado todos los desafíos de esta etapa.',
      'return_map': 'Volver al mapa',
      'well_done': '¡Bien hecho!',
      'i_completed': 'He completado esto',
      'level_up_suggestion_title': 'Sugerencia para subir de nivel',
      'level_up_suggestion_message':
          '¡Has completado 10 tareas en este nivel! Estás listo para el siguiente nivel. ¿Quieres avanzar?',
      'stay_here': 'Quédate aquí',
      'move_to_next_level': 'Pasar al siguiente nivel',
      // Reflection Screen
      'reflect': 'Reflexiona sobre tu crecimiento',
      'challenge': 'Desafío',
      'anxiety_q': '¿Qué tan ansioso te sentiste? (1-10)',
      'what_happened': '¿Qué pasó realmente?',
      'write_experience_hint': 'Escribe sobre tu experiencia...',
      'finish': 'Terminar reflexión',
      // General / Auth
      'welcome': 'Bienvenido a Bloom',
      'subtitle': 'Un espacio seguro para desarrollar tu confianza.',
      'start': 'Comenzar mi viaje',
      'guest': 'Continuar como invitado',
      'hello': 'Hola',
      'profile': 'Mi perfil',
      'history': 'Mi viaje de crecimiento',
      'streak': 'Racha actual',
      'best': 'Mejor racha',
      'points': 'Puntos de confianza',
      //Splash Screen
      'loading': 'Cargando tu jardín...',
    },
    'fr': {
      // Progress Screen
      'your_journey': 'Votre voyage',
      'keep_growing_sub':
          'Chaque petit pas est une victoire. Continuez à grandir !',
      'how_it_works': 'Comment ça marche ?',
      'total_points': 'Total de points',
      'current_streak': 'Série actuelle',
      'tasks_done': 'Tâches effectuées',
      'rank': 'Rang',
      'days': 'Jours',
      'contact_us': 'Contactez-nous',
      'contact_email_prompt':
          'Pour obtenir de l\'aide ou nous faire part de vos commentaires, veuillez nous envoyer un courriel à l\'adresse suivante :',
      'close': 'Fermer',
      // Level Map Screen
      'tap_to_view_journey': 'Appuyez para voir votre voyage ! 🌸',
      'tap_to_start': 'Appuyez pour commencer le défi',
      'choose_level': 'Choisissez votre étape de croissance :',
      'view_journey': 'Voir le voyage de croissance',
      'progress': 'Votre progression de croissance',
      'level_seedling': 'Plantule',
      'level_sprout': 'Pousse',
      'level_leaf': 'Feuille',
      'level_stem': 'Tige',
      'level_bloom': 'Fleur',
      // Profile Screen
      'account': 'Compte',
      'display_name': 'Nom d\'affichage',
      'save_name': 'Enregistrer le nom',
      'app_theme': 'Thème de l\'application',
      'select_color': 'Sélectionnez votre couleur Bloom :',
      'light_mode': 'Clair',
      'dark_mode': 'Sombre',
      'language': 'Langue',
      'logout': 'Déconnexion',
      'profile_updated': 'Profil mis à jour!',
      'pick_theme_color': 'Choisissez une couleur de thème',
      'done': 'Fait',
      'reset_all_progress': 'Réinitialiser toute la progression',
      'reset_confirm_title': 'Es-tu sûr?',
      'reset_confirm_message':
          'Cette action supprimera définitivement votre score de confiance, votre série et tout votre historique. Elle est irréversible.',
      'cancel': 'Annuler',
      'reset_everything': 'Tout réinitialiser',
      'reset_success': 'Toute la progression a été réinitialisée.',
      // Shop Screen
      'shop_title': 'Bloom Shop',
      'your_points': 'Vos points',
      'available_items': 'Articles disponibles',
      'streak_freeze': 'Gel de série',
      'protects_streak': 'Empêche la série de se réinitialiser',
      'your_inventory': 'Votre inventaire',
      'owned': 'Possédé',
      'equipped': 'Équipé',
      'equip_freeze': 'Équiper le gel',
      'buy': 'Acheter',
      'freeze_purchased': 'Geler l\'achat!',
      'not_enough_points': 'Pas assez de points!',
      'freeze_equipped': 'Congélateur équipé!',
      // Milestone Screen
      'your_growth_path': 'Votre chemin de croissance',
      'the_awakening': 'L\'éveil',
      'seed_badge': 'Badge Graine',
      'first_spark': 'Première étincelle',
      'bronze_leaf': 'Feuille de bronze',
      'social_courage': 'Courage social',
      'silver_sprout': 'Pousse d\'argent',
      'confidence_bloom': 'Floraison de confiance',
      'gold_flower': 'Fleur d\'or',
      'mastery': 'Maîtrise',
      'diamond_crown': 'Couronne de diamant',
      'milestone_claimed': 'Réclamé',
      'milestone_need_score': 'Besoin {0}%',
      'milestone_claim_reward': 'Réclamer +{0} pts',
      'milestone_reward_toast': 'Réclamé! +{0} pts',
      // FAQ Screen
      'help_faq': 'Aide et FAQ',
      'common_questions': 'Questions fréquentes',
      'keep_blooming': 'Continuez à fleurir ! 🌸',
      'faq_q1': 'Qu\'est-ce que Bloom ?',
      'faq_a1':
          'Bloom est un outil d\'auto-assistance conçu pour aider les gens à réduire l\'anxiété sociale grâce à un processus appelé \'Exposition Graduelle\'. En accomplissant de petites tâches sociales gérables, vous entraînez votre cerveau à réaliser que les interactions sociales sont sûres et surmontables.',
      'faq_q2': 'Comment fonctionnent les niveaux ?',
      'faq_a2':
          'Nous commençons par \'Jeune Pousse\' (tâches très faciles) et progressons jusqu\'à \'Floraison\' (tâches plus difficiles). À mesure que vous accomplissez des tâches, vous gagnez des points de confiance. Plus le niveau est élevé, plus vous gagnez de points !',
      'faq_q3': 'Qu\'est-ce que le Compteur de Confiance ?',
      'faq_a3':
          'La barre de progression sur votre écran d\'accueil représente votre confiance globale. Elle augmente au fur et à mesure que vous accomplissez des tâches. Attention : si vous arrêtez de vous entraîner pendant plusieurs jours, votre score de confiance peut légèrement diminuer, vous rappelant que la confiance est un muscle qui nécessite un exercice régulier !',
      'faq_q4': 'Qu\'est-ce qu\'une Série ?',
      'faq_a4':
          'Une série est le nombre de jours consécutifs où vous avez accompli au moins une tâche. La régularité est la clé pour surmonter l\'anxiété, alors essayez de garder votre flamme allumée !',
      'faq_q5': 'Où mes données sont-elles stockées ?',
      'faq_a5':
          'Votre vie privée est notre priorité. Tous vos progrès, votre historique et vos données de profil sont stockés localement sur votre propre appareil. Rien n\'est téléchargé sur un serveur cloud.',
      'faq_q6': 'Que faire si l\'application plante ?',
      'faq_a6':
          'Si l\'application se comporte de manière étrange, essayez de redémarrer votre téléphone. Si vous avez mis à jour l\'application, vous devrez peut-être vider le cache de l\'application dans vos paramètres Android. Si tout le reste échoue, vous pouvez utiliser l\'option \'Réinitialiser tous les progrès\' dans votre Profil.',
      'faq_q7': 'Puis-je sauter des niveaux ?',
      'faq_a7':
          'Oui ! Bien que nous recommandions le parcours progressif, vous êtes libre de choisir n\'importe quel niveau sur la carte qui correspond à votre niveau de confort actuel.',
      'faq_q8': 'Que faire si la tâche est très difficile ?',
      'faq_a8':
          'Bien que nous vous conseillons d\'essayer d\'accomplir la tâche, vous pouvez simplement revenir à l\'écran d\'accueil et y rentrer à nouveau pour changer la tâche actuelle.',
      'faq_q9': 'Contactez-nous',
      'faq_a9':
          'Nous serions ravis d\'avoir les retours de nos utilisateurs sur l\'application et de recevoir des suggestions pour les futures mises à jour. Nous aimerions savoir à quel point l\'application fonctionne bien pour les utilisateurs, ce qui lui manque et ce qui nécessite des améliorations. N\'hésitez pas à partager vos commentaires sur feedback.bloom@gmail.com, nous l\'apprécierions vraiment.',
      // Task Screen
      'stage_label': 'Étape : {0}',
      'keep_growing': 'Continuez à grandir, {0}',
      'current_challenge': 'Votre défi actuel pour cette étape :',
      'stage_mastered': 'Étape maîtrisée !',
      'all_done': 'Vous avez terminé tous les défis de cette étape.',
      'return_map': 'Retour à la carte',
      'well_done': 'Bien joué !',
      'i_completed': 'J\'ai terminé ceci',
      'level_up_suggestion_title': 'Suggestion de niveau supérieur',
      'level_up_suggestion_message':
          'Vous avez terminé 10 tâches en Ce niveau! Vous êtes prêt pour le niveau suivant. Voulez-vous passer au niveau supérieur ?',
      'stay_here': 'Reste ici',
      'move_to_next_level': 'Passer au niveau suivant',
      // Reflection Screen
      'reflect': 'Réfléchissez à votre croissance',
      'challenge': 'Défi',
      'anxiety_q': 'À quel point étiez-vous anxieux ? (1-10)',
      'what_happened': 'Que s\'est-il passé réellement ?',
      'write_experience_hint': 'Écrivez sur votre expérience...',
      'finish': 'Terminer la réflexion',
      // General / Auth
      'welcome': 'Bienvenue sur Bloom',
      'subtitle': 'Un espace sécurisé pour développer votre confiance.',
      'start': 'Commencer mon voyage',
      'guest': 'Continuer en tant qu\'invité',
      'hello': 'Bonjour',
      'profile': 'Mon profil',
      'history': 'Mon voyage de croissance',
      'streak': 'Série actuelle',
      'best': 'Meilleure série',
      'points': 'Points de confiance',
      //Splash Screen
      'loading': 'Chargement de votre jardin...',
    },
    'hi': {
      // Progress Screen
      'your_journey': 'आपकी यात्रा',
      'keep_growing_sub': 'हर छोटा कदम एक जीत है। आगे बढ़ते रहें!',
      'how_it_works': 'यह कैसे काम करता है?',
      'total_points': 'कुल अंक',
      'current_streak': 'वर्तमान लकीर',
      'tasks_done': 'कार्य पूर्ण',
      'rank': 'पद',
      'days': 'दिन',
      'contact_us': 'हमसे संपर्क करें',
      'contact_email_prompt':
          'सहायता और प्रतिक्रिया के लिए, हमें इस ईमेल पते पर ईमेल करें:',
      'close': 'बंद करना',
      // Level Map Screen
      'tap_to_view_journey': 'अपनी यात्रा देखने के लिए टैप करें! 🌸',
      'tap_to_start': 'चुनौती शुरू करने के लिए टैप करें',
      'choose_level': 'अपने विकास का चरण चुनें:',
      'view_journey': 'विकास यात्रा देखें',
      'progress': 'आपका विकास प्रोग्रेस',
      'level_seedling': 'अंकुर',
      'level_sprout': 'अंकुरण',
      'level_leaf': 'पत्ता',
      'level_stem': 'तना',
      'level_bloom': 'फूल',
      // Profile Screen
      'account': 'अकाउंट',
      'display_name': 'नाम',
      'save_name': 'नाम सेव करें',
      'app_theme': 'ऐप थीम',
      'select_color': 'अपना ब्लूम रंग चुनें:',
      'light_mode': 'लाइट मोड',
      'dark_mode': 'डार्क मोड',
      'language': 'भाषा',
      'logout': 'लॉग आउट',
      'profile_updated': 'प्रोफाइल अद्यतन किया गया!',
      'pick_theme_color': 'एक थीम रंग चुनें',
      'done': 'हो गया',
      'reset_all_progress': 'सभी प्रगति रीसेट करें',
      'reset_confirm_title': 'क्या आपको यकीन है?',
      'reset_confirm_message':
          'इससे आपका कॉन्फिडेंस स्कोर, स्ट्रीक और सारा इतिहास हमेशा के लिए डिलीट हो जाएगा। इसे वापस नहीं लाया जा सकता।',
      'cancel': 'रद्द करना',
      'reset_everything': 'सब कुछ रीसेट करें',
      'reset_success': 'सभी प्रगति रीसेट हो गई है।',
      // Shop Screen
      'shop_title': 'ब्लूम शॉप',
      'your_points': 'आपके पॉइंट्स',
      'available_items': 'उपलब्ध आइटम',
      'streak_freeze': 'स्ट्र्रीक फ्रीज',
      'protects_streak': 'स्ट्र्रीक को रीसेट होने से बचाता है',
      'your_inventory': 'आपकी इन्वेंट्री',
      'owned': 'प्राप्त किया',
      'equipped': 'इस्तेमाल में',
      'equip_freeze': 'फ्रीज का उपयोग करें',
      'buy': 'खरीदें',
      'freeze_purchased': 'फ़्रीज़ खरीदा गया!',
      'not_enough_points': 'पर्याप्त अंक नहीं!',
      'freeze_equipped': 'फ्रीज सुसज्जित!',
      // Milestone Screen
      'your_growth_path': 'आपका विकास पथ',
      'the_awakening': 'जागृति (The Awakening)',
      'seed_badge': 'बीज बैज (Seed Badge)',
      'first_spark': 'पहली चिंगारी (First Spark)',
      'bronze_leaf': 'कांस्य पत्ता (Bronze Leaf)',
      'social_courage': 'सामाजिक साहस (Social Courage)',
      'silver_sprout': 'चांदी का अंकुर (Silver Sprout)',
      'confidence_bloom': 'आत्मविश्वास का खिलना',
      'gold_flower': 'स्वर्ण फूल (Gold Flower)',
      'mastery': 'महारत (Mastery)',
      'diamond_crown': 'हीरे का मुकुट (Diamond Crown)',
      'milestone_claimed': 'दावा किया',
      'milestone_need_score': 'ज़रूरत {0}%',
      'milestone_claim_reward': 'दावा +{0} pts',
      'milestone_reward_toast': 'दावा किया! +{0} pts',
      // FAQ Screen
      'help_faq': 'सहायता और अक्सर पूछे जाने वाले प्रश्न',
      'common_questions': 'सामान्य प्रश्न',
      'keep_blooming': 'खिलते रहें! 🌸',
      'faq_q1': 'ब्लूम क्या है?',
      'faq_a1':
          'ब्लूम एक सेल्फ-हेल्प टूल है जिसे \'ग्रेडेड एक्सपोज़र\' नामक प्रक्रिया के माध्यम से लोगों की सामाजिक घबराहट (सोशल एंग्जायटी) को कम करने में मदद करने के लिए डिज़ाइन किया गया है। छोटे और आसान सामाजिक कार्यों को पूरा करके, आप अपने दिमाग को यह सिखाते हैं कि सामाजिक बातचीत पूरी तरह से सुरक्षित और सामान्य हैं।',
      'faq_q2': 'लेवल कैसे काम करते हैं?',
      'faq_a2':
          'हम \'सीडलिंग\' (बहुत आसान काम) से शुरुआत करते हैं और \'ब्लूम\' (अधिक चुनौतीपूर्ण काम) तक जाते हैं। जैसे-जैसे आप काम पूरा करते हैं, आप आत्मविश्वास पॉइंट्स कमाते हैं। लेवल जितना ऊंचा होगा, आप उतने ही अधिक पॉइंट्स कमाएंगे!',
      'faq_q3': 'कॉन्फिडेंस मीटर क्या है?',
      'faq_a3':
          'आपकी होम स्क्रीन पर प्रोग्रेस बार आपके समग्र आत्मविश्वास को दर्शाता है। जैसे-जैसे आप काम पूरा करते हैं, यह बढ़ता जाता है। ध्यान रखें: यदि आप कई दिनों तक अभ्यास बंद कर देते हैं, तो आपका आत्मविश्वास स्कोर थोड़ा कम हो सकता है, जो आपको याद दिलाता है कि आत्मविश्वास एक मांसपेशी की तरह है जिसे नियमित व्यायाम की आवश्यकता होती है!',
      'faq_q4': 'स्ट्र्रीक क्या है?',
      'faq_a4':
          'स्ट्र्रीक इस बात की गिनती है कि आपने लगातार कितने दिनों तक कम से कम एक काम पूरा किया है। घबराहट पर काबू पाने के लिए निरंतरता ही मुख्य चाबी है, इसलिए अपनी लपट को जलाए रखने का प्रयास करें!',
      'faq_q5': 'मेरा डेटा कहाँ स्टोर होता है?',
      'faq_a5':
          'आपकी प्राइवेसी हमारी प्राथमिकता है। आपकी सभी प्रोग्रेस, इतिहास और प्रोफाइल डेटा स्थानीय रूप से आपके अपने डिवाइस पर स्टोर किया जाता है। क्लाउड सर्वर पर कुछ भी अपलोड नहीं किया जाता है।',
      'faq_q6': 'यदि ऐप क्रैश हो जाए तो मैं क्या करूँ?',
      'faq_a6':
          'यदि ऐप ठीक से काम नहीं कर रहा है, तो अपना फोन रीस्टार्ट करने का प्रयास करें। यदि आपने ऐप को अपडेट किया है, तो आपको अपने एंड्रॉइड सेटिंग्स में ऐप कैशे (Cache) को क्लियर करना पड़ सकता है। यदि कुछ भी काम न करे, तो आप अपनी प्रोफाइल में \'सभी प्रोग्रेस रीसेट करें\' विकल्प का उपयोग कर सकते हैं।',
      'faq_q7': 'क्या मैं लेवल छोड़ (Skip) सकता हूँ?',
      'faq_a7':
          'हाँ! हालाँकि हम क्रमिक पथ की अनुशंसा करते हैं, आप मैप से कोई भी लेवल चुनने के लिए स्वतंत्र हैं जो आपके वर्तमान कम्फर्ट लेवल के लिए उपयुक्त हो।',
      'faq_q8': 'यदि कोई काम बहुत कठिन हो तो क्या होगा?',
      'faq_a8':
          'हालाँकि हम आपको काम पूरा करने की कोशिश करने की सलाह देते हैं, लेकिन आप केवल होम स्क्रीन पर वापस जा सकते हैं, और वर्तमान काम को बदलने के लिए फिर से प्रवेश कर सकते हैं।',
      'faq_q9': 'हमसे संपर्क करें',
      'faq_a9':
          'हम अपने उपयोगकर्ताओं से ऐप के बारे में सुनना और भविष्य के अपडेट के लिए सुझाव प्राप्त करना पसंद करेंगे। हमें यह जानकर खुशी होगी कि ऐप उपयोगकर्ताओं के लिए कितना अच्छा काम करता है, और इसमें किस चीज़ की कमी है और कहाँ सुधार की आवश्यकता है। कृपया feedback.bloom@gmail.com पर प्रतिक्रिया साझा करने में संकोच न करें, हम वास्तव में इसकी सराहना करेंगे।',
      // Task Screen
      'stage_label': 'चरण: {0}',
      'keep_growing': 'आगे बढ़ते रहें, {0}',
      'current_challenge': 'इस चरण के लिए आपकी वर्तमान चुनौती:',
      'stage_mastered': 'चरण पूरा हुआ!',
      'all_done': 'आपने इस चरण की सभी चुनौतियाँ पूरी कर ली हैं।',
      'return_map': 'मैप पर वापस जाएं',
      'well_done': 'बहुत बढ़िया!',
      'i_completed': 'मैंने यह पूरा कर लिया',
      'level_up_suggestion_title': 'स्तर बढ़ाने का सुझाव',
      'level_up_suggestion_message':
          'आपने इ हद में 10 कार्य पूरे कर लिए हैं! आप अगले स्तर के लिए तैयार हैं। क्या आप आगे बढ़ना चाहते हैं?',
      'stay_here': 'यहाँ रहें',
      'move_to_next_level': 'अगले स्तर पर जाएँ',
      // Reflection Screen
      'reflect': 'अपने विकास पर विचार करें',
      'challenge': 'चुनौती',
      'anxiety_q': 'आपने कितनी घबराहट महसूस की? (1-10)',
      'what_happened': 'वास्तव में क्या हुआ था?',
      'write_experience_hint': 'अपने अनुभव के बारे में लिखें...',
      'finish': 'विचार पूरा करें',
      // General / Auth
      'welcome': 'ब्लूम (Bloom) में आपका स्वागत है',
      'subtitle': 'आत्मविश्वास बढ़ाने के लिए एक सुरक्षित जगह।',
      'start': 'अपनी यात्रा शुरू करें',
      'guest': 'गेस्ट के रूप में आगे बढ़ें',
      'hello': 'नमस्ते',
      'profile': 'मेरी प्रोफाइल',
      'history': 'मेरी विकास यात्रा',
      'streak': 'करंट स्ट्र्रीक',
      'best': 'सर्वश्रेष्ठ स्ट्र्रीक',
      'points': 'आत्मविश्वास पॉइंट्स',
      //Splash Screen
      'loading': 'आपका बगीचा लोड हो रहा है...',
    },
    'de': {
      // Progress Screen
      'your_journey': 'Deine Reise',
      'keep_growing_sub': 'Jeder kleine Schritt ist ein Sieg. Wachse weiter!',
      'how_it_works': 'Wie funktioniert es?',
      'total_points': 'Gesamtpunktzahl',
      'current_streak': 'Aktueller Streak',
      'tasks_done': 'Aufgaben erledigt',
      'rank': 'Rang',
      'days': 'Tage',
      'contact_us': 'Kontaktieren Sie uns',
      'contact_email_prompt':
          'Für Unterstützung und Feedback senden Sie uns bitte eine E-Mail an:',
      'close': 'Schließen',
      // Level Map Screen
      'tap_to_view_journey': 'Tippe hier, um deine Reise zu sehen! 🌸',
      'tap_to_start': 'Tippe hier, um die Herausforderung zu starten',
      'choose_level': 'Wähle deine Entwicklungsstufe:',
      'view_journey': 'Entwicklungsreise anzeigen',
      'progress': 'Dein Entwicklungsfortschritt',
      'level_seedling': 'Keimling',
      'level_sprout': 'Spross',
      'level_leaf': 'Blatt',
      'level_stem': 'Stiel',
      'level_bloom': 'Blüte',
      // Profile Screen
      'account': 'Konto',
      'display_name': 'Anzeigename',
      'save_name': 'Name speichern',
      'app_theme': 'App-Design',
      'select_color': 'Wähle deine Bloom-Farbe:',
      'light_mode': 'Hell',
      'dark_mode': 'Dunkel',
      'language': 'Sprache',
      'logout': 'Abmelden',
      'profile_updated': 'Profil aktualisiert!',
      'pick_theme_color': 'Wählen Sie eine Designfarbe',
      'done': 'Erledigt',
      'reset_all_progress': 'Setzen Sie den gesamten Fortschritt zurück',
      'reset_confirm_title': 'Bist du sicher?',
      'reset_confirm_message':
          'Dadurch werden dein Konfidenzwert, deine Serie und dein gesamter Verlauf endgültig gelöscht. Dieser Vorgang kann nicht rückgängig gemacht werden.',
      'cancel': 'Stornieren',
      'reset_everything': 'Alles zurücksetzen',
      'reset_success': 'Alle Fortschritte wurden zurückgesetzt.',
      // Shop Screen
      'shop_title': 'Bloom Shop',
      'your_points': 'Deine Punkte',
      'available_items': 'Verfügbare Artikel',
      'streak_freeze': 'Streak-Freeze',
      'protects_streak': 'Schützt deine Strähne vor dem Zurücksetzen',
      'your_inventory': 'Dein Inventar',
      'owned': 'Besessen',
      'equipped': 'Ausrüstet',
      'equip_freeze': 'einfrieren ausrüsten',
      'buy': 'Kaufen',
      'freeze_purchased': 'Gekauft einfrieren!',
      'not_enough_points': 'Nicht genug Punkte!',
      'freeze_equipped': 'Einfrieren ausgestattet!',
      // Milestone Screen
      'your_growth_path': 'Dein Entwicklungspfad',
      'the_awakening': 'Das Erwachen',
      'seed_badge': 'Samen-Abzeichen',
      'first_spark': 'Erster Funke',
      'bronze_leaf': 'Bronzeblatt',
      'social_courage': 'Sozialer Mut',
      'silver_sprout': 'Silberspross',
      'confidence_bloom': 'Selbstvertrauen-Blüte',
      'gold_flower': 'Goldblume',
      'mastery': 'Meisterschaft',
      'diamond_crown': 'Diamantenkrone',
      'milestone_claimed': 'Behauptet',
      'milestone_need_score': 'Brauchen {0}%',
      'milestone_claim_reward': 'Beanspruchen +{0} pts',
      'milestone_reward_toast': 'Behauptet! +{0} pts',
      // FAQ Screen
      'help_faq': 'Hilfe und FAQs',
      'common_questions': 'Häufige Fragen',
      'keep_blooming': 'Blühe weiter! 🌸',
      'faq_q1': 'Was ist Bloom?',
      'faq_a1':
          'Bloom ist ein Selbsthilfewerkzeug, das entwickelt wurde, um Menschen dabei zu helfen, soziale Ängste durch einen Prozess namens \'Abgestufte Exposition\' zu reduzieren. Durch das Abschließen kleiner, bewältigbarer sozialer Aufgaben trainierst du dein Gehirn zu erkennen, dass soziale Interaktionen sicher und machbar sind.',
      'faq_q2': 'Wie funktionieren die Stufen?',
      'faq_a2':
          'Wir beginnen mit \'Keimling\' (sehr einfachen Aufgaben) und arbeiten uns hoch zu \'Blüte\' (anspruchsvolleren Aufgaben). Wenn du Aufgaben abschließt, verdienst du Selbstvertrauen-Punkte. Je höher die Stufe, desto mehr Punkte verdienst du!',
      'faq_q3': 'Was ist der Confidencemeter?',
      'faq_a3':
          'Der Fortschrittsbalken auf deinem Startbildschirm stellt dein gesamtes Selbstvertrauen dar. Er wächst, wenn du Aufgaben erledigst. Aber pass auf: Wenn du mehrere Tage lang nicht übst, kann dein Selbstvertrauenswert leicht sinken. Das erinnert dich daran, dass Selbstvertrauen ein Muskel ist, der regelmäßiges Training braucht!',
      'faq_q4': 'Was ist eine Strähne (Streak)?',
      'faq_a4':
          'Eine Strähne ist die Anzahl der aufeinanderfolgenden Tage, an denen du mindestens eine Aufgabe abgeschlossen hast. Beständigkeit ist der Schlüssel zur Überwindung von Ängsten, also versuche, deine Flamme am Brennen zu halten!',
      'faq_q5': 'Wo werden meine Daten gespeichert?',
      'faq_a5':
          'Deine Privatsphäre ist unsere Priorität. Dein gesamter Fortschritt, dein Verlauf und deine Profildaten werden lokal auf deinem eigenen Gerät gespeichert. Es wird nichts auf einen Cloud-Server hochgeladen.',
      'faq_q6': 'Was mache ich, wenn die App abstürzt?',
      'faq_a6':
          'Wenn sich die App seltsam verhält, versuche dein Telefon neu zu starten. Wenn du die App aktualisiert hast, musst du möglicherweise den App-Cache in deinen Android-Einstellungen leeren. Wenn alles andere fehlschlägt, kannst du die Option \'Gesamten Fortschritt zurücksetzen\' in deinem Profil nutzen.',
      'faq_q7': 'Kann ich Stufen überspringen?',
      'faq_a7':
          'Ja! Obwohl wir den schrittweisen Weg empfehlen, steht es dir frei, jede Stufe auf der Karte zu wählen, die sich für dein aktuelles Wohlbefinden richtig anfühlt.',
      'faq_q8': 'Was ist, wenn die Aufgabe sehr schwer ist?',
      'faq_a8':
          'Obwohl wir dir empfehlen, zu versuchen, die Aufgabe zu bewältigen, kannst du einfach zum Startbildschirm zurückkehren und erneut hineingehen, um die aktuelle Aufgabe zu ändern.',
      'faq_q9': 'Kontaktiere uns',
      'faq_a9':
          'Wir würden uns freuen, von unseren Nutzern Feedback zu unserer App sowie Empfehlungen für zukünftige Updates zu erhalten. Wir möchten erfahren, wie gut die App für die Nutzer funktioniert, was ihr fehlt und was verbessert werden muss. Bitte zögere nicht, dein Feedback unter feedback.bloom@gmail.com zu teilen, wir würden uns sehr darüber freuen.',
      // Task Screen
      'stage_label': 'Stufe: {0}',
      'keep_growing': 'Wachse weiter, {0}',
      'current_challenge': 'Deine aktuelle Herausforderung für diese Stufe:',
      'stage_mastered': 'Stufe gemeistert!',
      'all_done':
          'Du hast alle Herausforderungen auf dieser Stufe abgeschlossen.',
      'return_map': 'Zurück zur Karte',
      'well_done': 'Gut gemacht!',
      'i_completed': 'Ich habe das geschafft',
      'level_up_suggestion_title': 'Vorschlag zum Levelaufstieg',
      'level_up_suggestion_message':
          'Du hast 10 Aufgaben in Dieses Level abgeschlossen! Du bist bereit für das nächste Level. Möchtest du aufsteigen?',
      'stay_here': 'Bleib hier',
      'move_to_next_level': 'Gehen Sie zur nächsten Ebene',
      // Reflection Screen
      'reflect': 'Reflektiere über deine Entwicklung',
      'challenge': 'Herausforderung',
      'anxiety_q': 'Wie nervös hast du dich gefühlt? (1-10)',
      'what_happened': 'Was ist wirklich passiert?',
      'write_experience_hint': 'Schreibe über deine Erfahrung...',
      'finish': 'Reflektion beenden',
      // General / Auth
      'welcome': 'Willkommen bei Bloom',
      'subtitle': 'Ein sicherer Ort, um dein Selbstvertrauen zu stärken.',
      'start': 'Meine Reise starten',
      'guest': 'Als Gast fortfahren',
      'hello': 'Hallo',
      'profile': 'Mein Profil',
      'history': 'Meine Entwicklungsreise',
      'streak': 'Aktuelle Strähne',
      'best': 'Beste Strähne',
      'points': 'Selbstvertrauen-Punkte',
      //Splash Screen
      'loading': 'Ihr Garten wird geladen...',
    },
    'ur': {
      // Progress Screen
      'your_journey': 'آپ کا سفر',
      'keep_growing_sub': 'ہر چھوٹا قدم ایک فتح ہے۔ آگے بڑھتے رہیں!',
      'how_it_works': 'یہ کیسے کام کرتا ہے؟',
      'total_points': 'کل پوائنٹس',
      'current_streak': 'موجودہ سٹریک',
      'tasks_done': 'مکمل شدہ کام',
      'rank': 'رینک',
      'days': 'دن',
      'contact_us': 'ہم سے رابطہ کریں',
      'contact_email_prompt':
          'سپورٹ اور فیڈ بیک کے لیے، ہمیں اس ای میل پر رابطہ کریں:',
      'close': 'بند کریں',
      // Level Map Screen
      'tap_to_view_journey': 'اپنا سفر دیکھنے کے لیے ٹیپ کریں! 🌸',
      'tap_to_start': 'چیلنج شروع کرنے کے لیے ٹیپ کریں',
      'choose_level': 'اپنے ارتقاء کا مرحلہ منتخب کریں:',
      'view_journey': 'ارتقائی سفر دیکھیں',
      'progress': 'آپ کی ارتقائی ترقی',
      'level_seedling': 'پودا',
      'level_sprout': 'کونپل',
      'level_leaf': 'پتا',
      'level_stem': 'تنا',
      'level_bloom': 'شگوفہ',
      // Profile Screen
      'account': 'اکاؤنٹ',
      'display_name': 'ظاہری نام',
      'save_name': 'نام محفوظ کریں',
      'app_theme': 'ایپ تھیم',
      'select_color': 'اپنا بلوم رنگ منتخب کریں:',
      'light_mode': 'لائٹ',
      'dark_mode': 'ڈارک',
      'language': 'زبان',
      'logout': 'لاگ آؤٹ',
      'profile_updated': 'پروفائل اپ ڈیٹ ہو گئی!',
      'pick_theme_color': 'تھیم کا رنگ منتخب کریں',
      'done': 'مکمل',
      'reset_all_progress': 'تمام ترقی ری سیٹ کریں',
      'reset_confirm_title': 'کیا آپ کو یقین ہے؟',
      'reset_confirm_message':
          'اس سے آپ کا کانفیڈنس اسکور، سٹریک اور تمام ہسٹری مستقل طور پر حذف ہو جائے گی۔ اسے واپس نہیں لایا جا سکتا۔',
      'cancel': 'منسوخ کریں',
      'reset_everything': 'سب کچھ ری سیٹ کریں',
      'reset_success': 'تمام ترقی ری سیٹ کر دی گئی ہے۔',
      // Shop Screen
      'shop_title': 'بلوم شاپ',
      'your_points': 'آپ کے پوائنٹس',
      'available_items': 'دستیاب اشیاء',
      'streak_freeze': 'سٹریک فریز',
      'protects_streak': 'سٹریک کو ری سیٹ ہونے سے بچاتا ہے',
      'your_inventory': 'آپ کی انوینٹری',
      'owned': 'ملکیت میں',
      'equipped': 'لگا ہوا',
      'equip_freeze': 'فریز لگی ہے',
      'buy': 'خریدیں',
      'freeze_purchased': 'فریز خرید لی گئی ہے!',
      'not_enough_points': 'کافی پوائنٹس نہیں ہیں!',
      'freeze_equipped': 'فریز لگا دی گئی ہے!',
      // Milestone Screen
      'your_growth_path': 'آپ کا ارتقائی راستہ',
      'the_awakening': 'بیدار ہونا',
      'seed_badge': 'بیج کا بیج',
      'first_spark': 'پہلی چنگاری',
      'bronze_leaf': 'کانسی کا پتا',
      'social_courage': 'سماجی ہمت',
      'silver_sprout': 'چاندی کی کونپل',
      'confidence_bloom': 'اعتماد کا کھلنا',
      'gold_flower': 'سونے کا پھول',
      'mastery': 'مہارت',
      'diamond_crown': 'ہیرے کا تاج',
      'milestone_claimed': 'دعویٰ کیا۔',
      'milestone_need_score': 'ضرورت ہے {0}%',
      'milestone_claim_reward': 'دعویٰ +{0} pts',
      'milestone_reward_toast': 'دعوی کیا! +{0} pts',
      // FAQ Screen
      'help_faq': 'مدد اور عمومی سوالات',
      'common_questions': 'عام سوالات',
      'keep_blooming': 'کھلتے رہیں! 🌸',
      'faq_q1': 'بلوم کیا ہے؟',
      'faq_a1':
          'بلوم ایک سیلف ہیلپ ٹول ہے جو لوگوں کی سماجی گھبراہٹ (سوشل اینگزائٹی) کو کم کرنے کے لیے ڈیزائن کیا گیا ہے، اس عمل کو \'گریڈڈ ایکسپوژر\' کہا جاتا ہے۔ چھوٹے اور آسان سماجی کاموں کو مکمل کر کے، آپ اپنے دماغ کو یہ سکھاتے ہیں کہ سماجی روابط محفوظ اور قابل انتظام ہیں۔',
      'faq_q2': 'لیولز کیسے کام کرتے ہیں؟',
      'faq_a2':
          'ہم \'پودا\' (بہت آسان کاموں) سے شروع کرتے ہیں اور \'بلوم\' (زیادہ چیلنجنگ کاموں) تک جاتے ہیں۔ جیسے جیسے آپ کام مکمل کرتے ہیں، آپ اعتماد کے پوائنٹس حاصل کرتے ہیں۔ لیول جتنا اونچا ہوگا، آپ اتنے ہی زیادہ پوائنٹس کمائیں گے!',
      'faq_q3': 'کانفیڈنس میٹر کیا ہے؟',
      'faq_a3':
          'آپ کی ہوم اسکرین پر موجود پروگریس بار آپ کے مجموعی اعتماد کی عکاسی کرتا ہے۔ یہ کاموں کو مکمل کرنے کے ساتھ ساتھ بڑھتا ہے۔ ہوشیار رہیں: اگر آپ کئی دنوں تک مشق بند کر دیتے ہیں، تو آپ کا کانفیڈنس اسکور تھوڑا کم ہو سکتا ہے، جو آپ کو یاد دلاتا ہے کہ اعتماد ایک مسل کی طرح ہے جسے باقاعدہ ورزش کی ضرورت ہوتی ہے!',
      'faq_q4': 'سٹریک کیا ہے؟',
      'faq_a4':
          'سٹریک اس بات کی گنتی ہے کہ آپ نے لگاتار کتنے دن کم از کم ایک کام مکمل کیا ہے۔ گھبراہٹ پر قابو پانے کے لیے مستقل مزاجی ہی بنیادی ضرورت ہے، اس لیے اپنی شمع کو جلاتے رکھنے کی کوشش کریں!',
      'faq_q5': 'میرا ڈیٹا کہاں محفوظ ہوتا ہے؟',
      'faq_a5':
          'آپ کی پرائیویسی ہماری ترجیح ہے۔ آپ کی تمام ترقی، ہسٹری اور پروفائل ڈیٹا مقامی طور پر آپ کے اپنے ڈیوائس پر محفوظ کیا جاتا ہے۔ کلاؤڈ سرور پر کچھ بھی اپ لوڈ نہیں کیا جاتا۔',
      'faq_q6': 'اگر ایپ کریش ہو جائے تو میں کیا کروں؟',
      'faq_a6':
          'اگر ایپ کا رویہ عجیب ہو تو اپنا فون ری اسٹارٹ کرنے کی کوشش کریں۔ اگر آپ نے ایپ اپ ڈیٹ کی ہے، تو آپ کو اپنی اینڈرائیڈ سیٹنگز میں ایپ کیشے (Cache) کلیئر کرنے کی ضرورت پڑ سکتی ہے۔ اگر کچھ بھی کام نہ کرے، تو آپ اپنی پروفائل میں \'تمام ترقی ری سیٹ کریں\' کا آپشن استعمال کر سکتے ہیں۔',
      'faq_q7': 'کیا میں لیولز چھوڑ (Skip) سکتا ہوں؟',
      'faq_a7':
          'جی ہاں! اگرچہ ہم بتدریج راستے کی سفارش کرتے ہیں، لیکن آپ نقشے سے کوئی بھی لیول منتخب کرنے کے لیے آزاد ہیں جو آپ کے موجودہ کمفرٹ لیول کے مطابق ہو۔',
      'faq_q8': 'اگر کام بہت مشکل ہو تو کیا ہوگا؟',
      'faq_a8':
          'اگرچہ ہم آپ کو کام مکمل کرنے کی کوشش کرنے کا مشورہ دیتے ہیں، لیکن آپ صرف ہوم اسکرین پر واپس جا سکتے ہیں، اور موجودہ کام کو تبدیل کرنے کے لیے دوبارہ داخل ہو سکتے ہیں۔',
      'faq_q9': 'ہم سے رابطہ کریں',
      'faq_a9':
          'ہم اپنے صارفین سے ایپ کے بارے میں سننا اور مستقبل کی اپ ڈیٹس کے لیے تجاویز حاصل کرنا پسند کریں گے۔ ہمیں یہ جان کر خوشی ہوگی کہ ایپ صارفین کے لیے کتنی اچھی کارکردگی دکھا رہی ہے، اور اس میں کس چیز کی کمی ہے اور کہاں بہتری کی ضرورت ہے۔ براہ کرم feedback.bloom@gmail.com پر اپنی رائے شیئر کرنے میں ہچکچاہٹ محسوس نہ کریں، ہم واقعی اس کی تعریف کریں گے۔',
      // Task Screen
      'stage_label': 'مرحلہ: {0}',
      'keep_growing': 'آگے بڑھتے رہیں، {0}',
      'current_challenge': 'اس مرحلے کے لیے آپ کا موجودہ چیلنج:',
      'stage_mastered': 'مرحلہ مکمل ہوا!',
      'all_done': 'آپ نے اس مرحلے کے تمام چیلنجز مکمل کر لیے ہیں۔',
      'return_map': 'نقشے پر واپس جائیں',
      'well_done': 'بہت خوب!',
      'i_completed': 'میں نے یہ مکمل کر لیا',
      'level_up_suggestion_title': 'لیول اپ کی تجویز',
      'level_up_suggestion_message':
          'آپ نے اس سطح میں 10 کام مکمل کیے ہیں! آپ اگلے درجے کے لیے تیار ہیں۔ اوپر جانا چاہتے ہیں؟',
      'stay_here': 'یہیں رہو',
      'move_to_next_level': 'اگلی سطح پر جائیں۔',
      // Reflection Screen
      'reflect': 'اپنی ترقی پر غور کریں',
      'challenge': 'چیلنج',
      'anxiety_q': 'آپ نے کتنی گھبراہٹ محسوس کی؟ (1-10)',
      'what_happened': 'اصل میں کیا ہوا تھا؟',
      'write_experience_hint': 'اپنے تجربے کے بارے میں لکھیں...',
      'finish': 'غور و فکر مکمل کریں',
      // General / Auth
      'welcome': 'بلوم میں خوش آمدید',
      'subtitle': 'آپ کے اعتماد کو بڑھانے کے لیے ایک محفوظ جگہ۔',
      'start': 'میرا سفر شروع کریں',
      'guest': 'بطور مہمان جاری رکھیں',
      'hello': 'ہیلو',
      'profile': 'میری پروفائل',
      'history': 'میرا ارتقائی سفر',
      'streak': 'موجودہ سٹریک',
      'best': 'بہترین سٹریک',
      'points': 'کانفیڈنس پوائنٹس',
      //Splash Screen
      'loading': 'آپ کا باغ لوڈ ہو رہا ہے...',
    },
    'ar': {
      // Progress Screen
      'your_journey': 'رحلتك',
      'keep_growing_sub': 'كل خطوة صغيرة هي انتصار. استمر في النمو!',
      'how_it_works': 'كيف يعمل؟',
      'total_points': 'إجمالي النقاط',
      'current_streak': 'النشاط المتتالي الحالي',
      'tasks_done': 'المهام المنجزة',
      'rank': 'المرتبة',
      'days': 'أيام',
      'contact_us': 'اتصل بنا',
      'contact_email_prompt':
          'للدعم وإبداء الملاحظات، راسلنا عبر البريد الإلكتروني على:',
      'close': 'إغلاق',
      // Level Map Screen
      'tap_to_view_journey': 'اضغط لعرض رحلتك! 🌸',
      'tap_to_start': 'اضغط لبدء التحدي',
      'choose_level': 'اختر مرحلة نموك:',
      'view_journey': 'عرض رحلة النمو',
      'progress': 'تقدم نموك',
      'level_seedling': 'الشتلة',
      'level_sprout': 'البرعم',
      'level_leaf': 'الورقة',
      'level_stem': 'الساق',
      'level_bloom': 'الازدهار',
      // Profile Screen
      'account': 'الحساب',
      'display_name': 'الاسم المستعار',
      'save_name': 'حفظ الاسم',
      'app_theme': 'مظهر التطبيق',
      'select_color': 'اختر لون الازدهار الخاص بك:',
      'light_mode': 'فاتح',
      'dark_mode': 'داكن',
      'language': 'اللغة',
      'logout': 'تسجيل الخروج',
      'profile_updated': 'تم تحديث الملف الشخصي!',
      'pick_theme_color': 'اختر لون المظهر',
      'done': 'تم',
      'reset_all_progress': 'إعادة تعيين كل التقدم',
      'reset_confirm_title': 'هل أنت متأكد؟',
      'reset_confirm_message':
          'سيؤدي هذا إلى حذف درجة ثقتك ونشاطك المتتالي وسجلك بالكامل بشكل دائم. لا يمكن التراجع عن هذا الإجراء.',
      'cancel': 'إلغاء',
      'reset_everything': 'إعادة تعيين كل شيء',
      'reset_success': 'تمت إعادة تعيين كل التقدم.',
      // Shop Screen
      'shop_title': 'متجر بلوم',
      'your_points': 'نقاطك',
      'available_items': 'العناصر المتاحة',
      'streak_freeze': 'تجميد النشاط المتتالي',
      'protects_streak': 'يحمي النشاط المتتالي من إعادة التعيين',
      'your_inventory': 'مخزونك',
      'owned': 'تم الشراء',
      'equipped': 'مُفعّل',
      'equip_freeze': 'تفعيل التجميد',
      'buy': 'شراء',
      'freeze_purchased': 'تم شراء التجميد!',
      'not_enough_points': 'النقاط غير كافية!',
      'freeze_equipped': 'تم تفعيل التجميد!',
      // Milestone Screen
      'your_growth_path': 'مسار نموك',
      'the_awakening': 'الصحوة',
      'seed_badge': 'شارة البذرة',
      'first_spark': 'الشرارة الأولى',
      'bronze_leaf': 'الورقة البرونزية',
      'social_courage': 'الشجاعة الاجتماعية',
      'silver_sprout': 'البرعم الفضي',
      'confidence_bloom': 'ازدهار الثقة',
      'gold_flower': 'الزهرة الذهبية',
      'mastery': 'الإتقان',
      'diamond_crown': 'التاج الماسي',
      'milestone_claimed': 'تم الاستلام',
      'milestone_need_score': 'تحتاج {0}%',
      'milestone_claim_reward': 'استلام +{0} نقاط',
      'milestone_reward_toast': 'تم الاستلام! +{0} نقاط',
      // FAQ Screen
      'help_faq': 'المساعدة والأسئلة الشائعة',
      'common_questions': 'الأسئلة الشائعة',
      'keep_blooming': 'واصل الازدهار! 🌸',
      'faq_q1': 'ما هو بلوم؟',
      'faq_a1':
          'بلوم هو أداة للمساعدة الذاتية مصممة لمساعدة الأشخاص على تقليل القلق الاجتماعي من خلال عملية تسمى \'التعرض التدريجي\'. من خلال إكمال مهام اجتماعية صغيرة وقابلة للإدارة، فإنك تدرب عقلك على إدراك أن التفاعلات الاجتماعية آمنة وممكنة.',
      'faq_q2': 'كيف تعمل المستويات؟',
      'faq_a2':
          'نبدأ بـ \'الشتلة\' (مهام سهلة للغاية) وننتقل صعوداً إلى \'الازدهار\' (مهام أكثر تحدياً). عندما تكمل المهام، تكسب نقاط ثقة. كلما ارتفع المستوى، زادت النقاط التي تكسبها!',
      'faq_q3': 'ما هو مقياس الثقة؟',
      'faq_a3':
          'يمثل شريط التقدم على شاشتك الرئيسية ثقتك العامة. إنه ينمو كلما أكملت المهام. كن حذراً: إذا توقفت عن الممارسة لعدة أيام، فقد تنخفض درجة ثقتك قليلاً، مما يذكرك بأن الثقة عضلة تحتاج إلى تمرين منتظم!',
      'faq_q4': 'ما هو النشاط المتتالي (Streak)؟',
      'faq_a4':
          'النشاط المتتالي هو عدد الأيام المتتالية التي أكملت فيها مهمة واحدة على الأقل. الاستمرارية هي المفتاح للتغلب على القلق، لذا حاول أن تبقي شعلتك متقدة!',
      'faq_q5': 'أين يتم تخزين بياناتي؟',
      'faq_a5':
          'خصوصيتك هي أولويتنا. يتم تخزين كل تقدمك وسجلك وبيانات ملفك الشخصي محلياً على جهازك الخاص. لا يتم تحميل أي شيء إلى خادم سحابي.',
      'faq_q6': 'ماذا أفعل إذا تعطل التطبيق؟',
      'faq_a6':
          'إذا كان التطبيق يتصرف بشكل غريب، فحاول إعادة تشغيل هاتفك. إذا قمت بتحديث التطبيق، فقد تحتاج إلى مسح ذاكرة التخزين المؤقت للتطبيق في إعدادات أندرويد الخاصة بك. إذا فشل كل شيء آخر، يمكنك استخدام خيار \'إعادة تعيين كل التقدم\' في ملفك الشخصي.',
      'faq_q7': 'هل يمكنني تخطي المستويات؟',
      'faq_a7':
          'نعم! بينما نوصي بالمسار التدريجي، فأنت حر في اختيار أي مستوى من الخريطة يراه مناسباً لمستوى راحتك الحالي.',
      'faq_q8': 'ماذا لو كانت المهمة صعبة للغاية؟',
      'faq_a8':
          'بينما نوصيك بمحاولة إنجاز المهمة، يمكنك ببساطة العودة إلى الشاشة الرئيسية والدخول مجدداً لتغيير المهمة الحالية',
      'faq_q9': 'اتصل بنا',
      'faq_a9':
          'نود أن نسمع عن تطبيقنا من مستخدمينا، وتوصياتهم للتحديثات المستقبلية. نود أن نعرف مدى نجاح التطبيق مع المستخدمين، وما ينقصه ويحتاج إلى تحسين. لا تتردد في مشاركة الملاحظات على feedback.bloom@gmail.com، ونحن نقدر ذلك حقاً',
      // Task Screen
      'stage_label': 'المرحلة: {0}',
      'keep_growing': 'واصل النمو، {0}',
      'current_challenge': 'تحديك الحالي لهذه المرحلة:',
      'stage_mastered': 'تم إتقان المرحلة!',
      'all_done': 'لقد أكملت جميع التحديات في هذه المرحلة.',
      'return_map': 'العودة إلى الخريطة',
      'well_done': 'أحسنت صنعاً!',
      'i_completed': 'لقد أكملت هذا',
      'level_up_suggestion_title': 'اقتراح لرفع المستوى',
      'level_up_suggestion_message':
          'لقد أنجزت 10 مهام في هذا المستوى! أنت جاهز للمستوى التالي. هل تريد الانتقال إلى المستوى التالي؟?',
      'stay_here': 'البقاء هنا',
      'move_to_next_level': 'الانتقال إلى المستوى التالي',
      // Reflection Screen
      'reflect': 'تأمل في نموك',
      'challenge': 'تحدي',
      'anxiety_q': 'ما مدى القلق الذي شعرت به؟ (1-10)',
      'what_happened': 'ماذا حدث بالفعل؟',
      'write_experience_hint': 'اكتب عن تجربتك...',
      'finish': 'إنهاء التأمل',
      // General / Auth
      'welcome': 'مرحباً بك في بلوم',
      'subtitle': 'مساحة آمنة لتنمية ثقتك بنفسك.',
      'start': 'ابدأ رحلتي',
      'guest': 'المتابعة كضيف',
      'hello': 'مرحباً',
      'profile': 'ملفي الشخصي',
      'history': 'رحلة نموي',
      'streak': 'النشاط المتتالي الحالي',
      'best': 'أفضل نشاط متتالي',
      'points': 'نقاط الثقة',
      //Splash Screen
      'loading': 'جاري تحميل حديقتك...',
    },
    'ja': {
      // Progress Screen
      'your_journey': 'あなたの歩み',
      'keep_growing_sub': '小さな一歩がすべて勝利です。成長を続けましょう！',
      'how_it_works': 'どんな仕組み？',
      'total_points': '合計ポイント',
      'current_streak': '現在の継続日数',
      'tasks_done': '達成したクエスト',
      'rank': 'ランク',
      'days': '日',
      'contact_us': 'お問い合わせ',
      'contact_email_prompt': 'サポートやフィードバックについては、こちらにお問い合わせください：',
      'close': '閉じる',
      // Level Map Screen
      'tap_to_view_journey': 'タップしてあなたの歩みを見る！ 🌸',
      'tap_to_start': 'タップしてチャレンジを開始',
      'choose_level': '成長のステージを選択してください：',
      'view_journey': '成長の歩みを見る',
      'progress': 'あなたの成長進捗',
      'level_seedling': '苗木（Seedling）',
      'level_sprout': '新芽（Sprout）',
      'level_leaf': '若葉（Leaf）',
      'level_stem': '茎（Stem）',
      'level_bloom': '開花（Bloom）',
      // Profile Screen
      'account': 'アカウント',
      'display_name': '表示名',
      'save_name': '名前を保存',
      'app_theme': 'アプリのテーマ',
      'select_color': 'あなたの開花カラーを選択：',
      'light_mode': 'ライト',
      'dark_mode': 'ダーク',
      'language': '言語',
      'logout': 'ログアウト',
      'profile_updated': 'プロフィールを更新しました！',
      'pick_theme_color': 'テーマカラーを選択',
      'done': '完了',
      'reset_all_progress': 'すべての進捗をリセット',
      'reset_confirm_title': '本当に本当によろしいですか？',
      'reset_confirm_message':
          'これにより、自信スコア、継続日数、およびすべての履歴が完全に削除されます。この操作は取り消せません。',
      'cancel': 'キャンセル',
      'reset_everything': 'すべてをリセット',
      'reset_success': 'すべての進捗がリセットされました。',
      // Shop Screen
      'shop_title': 'ブルームショップ',
      'your_points': '所持ポイント',
      'available_items': '購入可能なアイテム',
      'streak_freeze': 'ストリーク・フリーズ',
      'protects_streak': '継続日数のリセットを防ぎます',
      'your_inventory': 'あなたのインベントリ',
      'owned': '所有中',
      'equipped': '装着中',
      'equip_freeze': 'フリーズを使用する',
      'buy': '購入',
      'freeze_purchased': 'フリーズを購入しました！',
      'not_enough_points': 'ポイントが足りません！',
      'freeze_equipped': 'フリーズを適用しました！',
      // Milestone Screen
      'your_growth_path': 'あなたの成長ロード',
      'the_awakening': '覚醒',
      'seed_badge': '種のバッジ',
      'first_spark': '最初のひらめき',
      'bronze_leaf': 'ブロンズの葉',
      'social_courage': '社交的な勇気',
      'silver_sprout': 'シルバーの芽',
      'confidence_bloom': '自信の開花',
      'gold_flower': 'ゴールドの花',
      'mastery': 'マスター',
      'diamond_crown': 'ダイヤモンドの王冠',
      'milestone_claimed': '獲得済み',
      'milestone_need_score': 'あと {0}% 必要',
      'milestone_claim_reward': '報酬を獲得 +{0} pts',
      'milestone_reward_toast': '獲得しました！ +{0} pts',
      // FAQ Screen
      'help_faq': 'ヘルプとFAQ',
      'common_questions': 'よくある質問',
      'keep_blooming': '咲き続けましょう！ 🌸',
      'faq_q1': 'Bloomとは何ですか？',
      'faq_a1':
          'Bloomは、「段階的暴露（スモールステップ）」と呼ばれるプロセスを通じて、社交不安を和らげるために設計されたセルフケアツールです。小さく管理しやすい社交的な課題をクリアしていくことで、脳に「人間関係は安全で対応可能である」ということを学習させます。',
      'faq_q2': 'レベルはどのように機能しますか？',
      'faq_a2':
          '「苗木（非常に簡単な課題）」から始まり、「開花（より挑戦的な課題）」へと進んでいきます。課題を達成するごと 自信ポイントを獲得できます。レベルが高くなるほど、より多くのポイントを獲得できます！',
      'faq_q3': '自信メーターとは何ですか？',
      'faq_a3':
          'ホーム画面のプログレスバーは、あなたの全体的な自信を表しています。課題を完了するごとに成長します。注意：数日間練習を休むと、自信スコアがわずかに減少することがあります。これは、自信が定期的な運動を必要とする筋肉のようなものであることを思い出させるためのものです！',
      'faq_q4': 'ストリーク（継続日数）とは何ですか？',
      'faq_a4':
          'ストリークは、少なくとも1つの課題を完了した連続日数のカウントです。継続こそが不安を克服する鍵ですので、心の炎を絶やさないようにしましょう！',
      'faq_q5': 'データはどこに保存されますか？',
      'faq_a5':
          'プライバシーの保護は私たちの最優先事項です。すべての進捗、履歴、プロフィールデータは、あなた自身のデバイス内にローカルに保存されます。クラウドサーバーにアップロードされることは一切ありません。',
      'faq_q6': 'アプリがクラッシュした場合はどうすればよいですか？',
      'faq_a6':
          'アプリの挙動がおかしい場合は、スマートフォンを再起動してみてください。アプリをアップデートした場合は、Androidの設定からアプリのキャッシュを消去する必要があるかもしれません。それでも解決しない場合は、プロフィールの「すべての進捗をリセット」オプションを使用できます。',
      'faq_q7': 'レベルをスキップすることはできますか？',
      'faq_a7':
          'はい！段階的なステップを踏むことをお勧めしますが、現在の快適さに合わせて、マップから任意のレベルを自由に選択していただけます。',
      'faq_q8': '課題が難しすぎる場合はどうすればよいですか？',
      'faq_a8':
          '課題を達成することをお勧めしますが、どうしても難しい場合は一度ホーム画面に戻り、再度入り直すことで現在の課題を変更することができます。',
      'faq_q9': 'お問い合わせ',
      'faq_a9':
          'ユーザーの皆様からのアプリに関するご意見や、今後のアップデートへのご要望をお待ちしております。アプリがどの程度役立っているか、何が不足していてどこに改善が必要かなど、ぜひお聞かせください。フィードバックは feedback.bloom@gmail.com までお気軽にお寄せください。心より感謝いたします。',
      // Task Screen
      'stage_label': 'ステージ: {0}',
      'keep_growing': 'その調子です、{0}さん',
      'current_challenge': 'このステージの現在のチャレンジ：',
      'stage_mastered': 'ステージクリア！',
      'all_done': 'このステージのすべてのチャレンジを完了しました。',
      'return_map': 'マップに戻る',
      'well_done': 'よくできました！',
      'i_completed': '達成しました',
      'level_up_suggestion_title': 'レベルアップの提案',
      'level_up_suggestion_message':
          'このレベルで10個のタスクを完了しました！次のレベルに進む準備ができています。レベルアップしたいですか？',
      'stay_here': 'ここにいてください',
      'move_to_next_level': '次のレベルに移動',
      // Reflection Screen
      'reflect': '自分の成長を振り返る',
      'challenge': 'チャレンジ',
      'anxiety_q': 'どのくらい不安を感じましたか？ (1-10)',
      'what_happened': '実際には何が起こりましたか？',
      'write_experience_hint': 'あなたの体験について書きましょう...',
      'finish': '振り返りを完了する',
      // General / Auth
      'welcome': 'Bloomへようこそ',
      'subtitle': '自信を育むための安全な場所。',
      'start': '旅を始める',
      'guest': 'ゲストとして続ける',
      'hello': 'こんにちは',
      'profile': 'マイプロフィール',
      'history': '私の成長の旅',
      'streak': '現在の継続日数',
      'best': '自己ベスト継続',
      'points': '自信ポイント',
      //Splash Screen
      'loading': '庭園を読み込んでいます...',
    },
    'ko': {
      // Progress Screen
      'your_journey': '나의 여정',
      'keep_growing_sub': '모든 작은 걸음이 승리입니다. 계속해서 성장해 나가세요!',
      'how_it_works': '어떻게 작동하나요?',
      'total_points': '총 포인트',
      'current_streak': '현재 연속 일수',
      'tasks_done': '완료한 미션',
      'rank': '랭크',
      'days': '일',
      'contact_us': '문의하기',
      'contact_email_prompt': '지원 및 피드백이 필요하시면 다음 이메일로 보내주세요:',
      'close': '닫기',
      // Level Map Screen
      'tap_to_view_journey': '탭하여 나의 여정을 확인하세요! 🌸',
      'tap_to_start': '탭하여 챌린지 시작',
      'choose_level': '성장 단계를 선택하세요:',
      'view_journey': '성장 여정 보기',
      'progress': '나의 성장 진척도',
      'level_seedling': '묘목 (Seedling)',
      'level_sprout': '새싹 (Sprout)',
      'level_leaf': '어린잎 (Leaf)',
      'level_stem': '줄기 (Stem)',
      'level_bloom': '개화 (Bloom)',
      // Profile Screen
      'account': '계정',
      'display_name': '닉네임',
      'save_name': '이름 저장',
      'app_theme': '앱 테마',
      'select_color': '나만의 블룸 색상 선택:',
      'light_mode': '라이트 모드',
      'dark_mode': '다크 모드',
      'language': '언어',
      'logout': '로그아웃',
      'profile_updated': '프로필이 업데이트되었습니다!',
      'pick_theme_color': '테마 색상 선택',
      'done': '완료',
      'reset_all_progress': '모든 진척도 리셋',
      'reset_confirm_title': '정말이신가요?',
      'reset_confirm_message':
          '이 작업은 회원님의 자신감 점수, 연속 일수 및 모든 히스토리를 영구적으로 삭제합니다. 취소할 수 없습니다.',
      'cancel': '취소',
      'reset_everything': '모든 것 리셋',
      'reset_success': '모든 진척도가 리셋되었습니다.',
      // Shop Screen
      'shop_title': '블룸 숍',
      'your_points': '보유 포인트',
      'available_items': '구매 가능한 아이템',
      'streak_freeze': '스트릭 프리즈',
      'protects_streak': '연속 일수가 초기화되는 것을 방지합니다',
      'your_inventory': '나의 인벤토리',
      'owned': '보유함',
      'equipped': '장착됨',
      'equip_freeze': '프리즈 장착하기',
      'buy': '구매',
      'freeze_purchased': '프리즈를 구매했습니다!',
      'not_enough_points': '포인트가 부족합니다!',
      'freeze_equipped': '프리즈가 장착되었습니다!',
      // Milestone Screen
      'your_growth_path': '나의 성장 경로',
      'the_awakening': '각성',
      'seed_badge': '씨앗 배지',
      'first_spark': '첫 번째 불꽃',
      'bronze_leaf': '브론즈 잎새',
      'social_courage': '사회적 용기',
      'silver_sprout': '실버 새싹',
      'confidence_bloom': '자신감의 개화',
      'gold_flower': '골드 플라워',
      'mastery': '마스터',
      'diamond_crown': '다이아몬드 크라운',
      'milestone_claimed': '수령 완료',
      'milestone_need_score': '{0}% 필요',
      'milestone_claim_reward': '보상 받기 +{0} pts',
      'milestone_reward_toast': '보상을 받았습니다! +{0} pts',
      // FAQ Screen
      'help_faq': '도움말 및 FAQ',
      'common_questions': '자주 묻는 질문',
      'keep_blooming': '계속해서 꽃을 피워내세요! 🌸',
      'faq_q1': 'Bloom은 어떤 앱인가요?',
      'faq_a1':
          'Bloom은 \'단계적 노출\'이라는 과정을 통해 사회적 불안감을 완화할 수 있도록 설계된 자가 치유 도구입니다. 작고 관리 가능한 수준의 사회적 미션들을 완수해 나감으로써, 사회적 상호작용이 안전하고 충분히 감당할 수 있는 것임을 뇌에 학습시킵니다.',
      'faq_q2': '레벨은 어떻게 구성되나요?',
      'faq_a2':
          '\'묘목\' 단계(매우 쉬운 미션)로 시작하여 \'개화\' 단계(더 도전적인 미션)로 올라갑니다. 미션을 완료할 때마다 자신감 포인트를 획득하며, 레벨이 높아질수록 더 많은 포인트를 얻을 수 있습니다!',
      'faq_q3': '자신감 미터란 무엇인가요?',
      'faq_a3':
          '홈 화면의 프로그레스 바는 회원님의 전반적인 자신감을 나타내며, 미션을 완료할 때마다 채워집니다. 주의해 주세요: 수일 동안 연습을 멈추면 자신감 점수가 약간 감소할 수 있습니다. 이는 자신감이 규칙적인 운동을 필요로 하는 근육과 같다는 것을 상기시켜 주기 위함입니다!',
      'faq_q4': '스트릭(연속 일수)이란 무엇인가요?',
      'faq_a4':
          '스트릭은 하루에 최소 하나 이상의 미션을 연속으로 완료한 일수를 나타냅니다. 불안감을 극복하는 열쇠는 꾸준함에 있으니, 회원님의 불꽃이 꺼지지 않도록 유지해 보세요!',
      'faq_q5': '내 데이터는 어디에 저장되나요?',
      'faq_a5':
          '회원님의 개인정보 보호가 저희의 최우선 과제입니다. 모든 진척도, 히스토리, 프로필 데이터는 외부 서버가 아닌 회원님의 기기에 로컬로만 저장됩니다.',
      'faq_q6': '앱이 크래시(강제 종료)되면 어떻게 하나요?',
      'faq_a6':
          '앱이 비정상적으로 작동하는 경우 휴대전화를 재부팅해 보세요. 앱을 업데이트한 후라면 안드로이드 설정에서 앱 캐시를 지워야 할 수도 있습니다. 모든 방법이 실패할 경우, 프로필 화면의 \'모든 진척도 리셋\' 옵션을 사용할 수 있습니다.',
      'faq_q7': '레벨을 건너뛸 수 있나요?',
      'faq_a7':
          '네, 가능합니다! 점진적인 단계를 따르는 것을 권장하지만, 현재 스스로 편안하게 느끼는 정도에 맞춰 지도(Map)상에서 원하는 레벨을 자유롭게 선택하실 수 있습니다.',
      'faq_q8': '미션이 너무 어렵게 느껴지면 어떻게 하나요?',
      'faq_a8':
          '미션을 완수하기 위해 노력해 보는 것을 권장하지만, 너무 어렵다면 단순히 홈 화면으로 돌아갔다가 다시 진입하여 현재 미션을 새로 바꿀 수 있습니다.',
      'faq_q9': '문의하기',
      'faq_a9':
          '저희는 사용자분들로부터 앱에 대한 소중한 의견이나 향후 업데이트를 위한 제안을 듣는 것을 언제나 환영합니다. 앱이 얼마나 도움이 되었는지, 어떤 점이 부족하고 개선이 필요한지 자유롭게 의견을 나누어 주세요. feedback.bloom@gmail.com 으로 피드백을 보내주시면 진심으로 감사하겠습니다.',
      // Task Screen
      'stage_label': '스테이지: {0}',
      'keep_growing': '잘하고 있어요, {0}님',
      'current_challenge': '이번 스테이지의 현재 챌린지:',
      'stage_mastered': '스테이지 마스터!',
      'all_done': '이번 스테이지의 모든 챌린지를 완료하셨습니다.',
      'return_map': '지도로 돌아가기',
      'well_done': '참 잘했어요!',
      'i_completed': '미션 완료함',
      'level_up_suggestion_title': '레벨업 제안',
      'level_up_suggestion_message':
          '이 수준에서 10개의 과제를 완료하셨습니다! 다음 레벨로 넘어갈 준비가 되셨나요? 더 높은 레벨로 올라가고 싶으신가요?',
      'stay_here': '여기에 머물러 라.',
      'move_to_next_level': '다음 레벨로 이동',
      // Reflection Screen
      'reflect': '나의 성장을 돌아보기',
      'challenge': '도전',
      'anxiety_q': '얼마나 불안하셨나요? (1-10)',
      'what_happened': '실제로는 어떤 일이 일어났나요?',
      'write_experience_hint': '그때의 경험에 대해 작성해 보세요...',
      'finish': '돌아보기 마칠게요',
      // General / Auth
      'welcome': 'Bloom에 오신 것을 환영합니다',
      'subtitle': '자신감을 키울 수 있는 안전한 공간.',
      'start': '나의 여정 시작하기',
      'guest': '게스트로 계속하기',
      'hello': '안녕하세요',
      'profile': '내 프로필',
      'history': '나의 성장 여정',
      'streak': '현재 연속 일수',
      'best': '최고 연속 일수',
      'points': '자신감 포인트',
      //Splash Screen
      'loading': '정원을 불러오는 중입니다...',
    },
    'zh': {
      // Progress Screen
      'your_journey': '你的旅程',
      'keep_growing_sub': '迈出的每一小步都是胜利。继续成长吧！',
      'how_it_works': '它是如何运作的？',
      'total_points': '总积分',
      'current_streak': '当前连续天数',
      'tasks_done': '已完成任务',
      'rank': '段位',
      'days': '天',
      'contact_us': '联系我们',
      'contact_email_prompt': '如需支持与反馈，请发送邮件至：',
      'close': '关闭',
      // Level Map Screen
      'tap_to_view_journey': '点击查看你的旅程！ 🌸',
      'tap_to_start': '点击开始挑战',
      'choose_level': '选择你的成长阶段：',
      'view_journey': '查看成长历程',
      'progress': '你的成长进度',
      'level_seedling': '幼苗 (Seedling)',
      'level_sprout': '新芽 (Sprout)',
      'level_leaf': '嫩叶 (Leaf)',
      'level_stem': '花茎 (Stem)',
      'level_bloom': '绽放 (Bloom)',
      // Profile Screen
      'account': '账户',
      'display_name': '昵称',
      'save_name': '保存昵称',
      'app_theme': '应用主题',
      'select_color': '选择你的绽放色彩：',
      'light_mode': '浅色',
      'dark_mode': '深色',
      'language': '语言',
      'logout': '退出登录',
      'profile_updated': '个人资料已更新！',
      'pick_theme_color': '选择主题颜色',
      'done': '完成',
      'reset_all_progress': '重置所有进度',
      'reset_confirm_title': '你确定吗？',
      'reset_confirm_message': '这将永久删除你的自信指数、连续天数和所有历史记录。此操作无法撤销。',
      'cancel': '取消',
      'reset_everything': '重置一切',
      'reset_success': '所有进度已重置。',
      // Shop Screen
      'shop_title': '绽放小铺',
      'your_points': '我的积分',
      'available_items': '可用道具',
      'streak_freeze': '连续天数冻结卡',
      'protects_streak': '保护连续天数不被重置',
      'your_inventory': '我的背包',
      'owned': '已拥有',
      'equipped': '已装备',
      'equip_freeze': '使用冻结卡',
      'buy': '购买',
      'freeze_purchased': '成功购买冻结卡！',
      'not_enough_points': '积分不足！',
      'freeze_equipped': '已成功装备冻结卡！',
      // Milestone Screen
      'your_growth_path': '你的成长之路',
      'the_awakening': '觉醒',
      'seed_badge': '种子徽章',
      'first_spark': '初次闪耀',
      'bronze_leaf': '青铜之叶',
      'social_courage': '社交勇气',
      'silver_sprout': '白银新芽',
      'confidence_bloom': '自信绽放',
      'gold_flower': '黄金之花',
      'mastery': '大师',
      'diamond_crown': '钻石皇冠',
      'milestone_claimed': '已领取',
      'milestone_need_score': '还需 {0}%',
      'milestone_claim_reward': '领取 +{0} 积分',
      'milestone_reward_toast': '领取成功！ +{0} 积分',
      // FAQ Screen
      'help_faq': '帮助与常见问题',
      'common_questions': '常见问题',
      'keep_blooming': '尽情绽放吧！ 🌸',
      'faq_q1': '什么是 Bloom？',
      'faq_a1':
          'Bloom 是一款自助心理工具，旨在通过一种被称为“系统脱敏（渐进式暴露）”的方法来帮助人们减轻社交焦虑。通过完成一个个微小且可控的社交任务，你将训练自己的大脑意识到：社交互动其实是安全并且能够应对的。',
      'faq_q2': '关卡是如何运作的？',
      'faq_a2':
          '我们从“幼苗”阶段（非常简单的任务）开始，逐步提升到“绽放”阶段（更具挑战性的任务）。每当你完成任务，就会获得自信积分。关卡越高，获得的积分就越多！',
      'faq_q3': '什么是自信指数计？',
      'faq_a3':
          '主屏幕上的进度条代表你整体的自信指数。它会随着你完成任务而增长。请注意：如果你连续几天停止练习，你的自信指数可能会略微衰退，以此提醒你——自信是一块需要定期锻炼的肌肉！',
      'faq_q4': '什么是连续天数（Streak）？',
      'faq_a4': '连续天数是指你连续有多少天完成了至少一项任务。持之以恒是战胜焦虑的关键，所以试着让你的成长火苗燃烧下去吧！',
      'faq_q5': '我的数据存储在哪里？',
      'faq_a5':
          '保护你的隐私是我们的重中之重。你所有的进度、历史记录和个人资料数据都完整保存在你的本地设备中。任何数据都不会被上传至云端服务器。',
      'faq_q6': '应用崩溃或异常怎么办？',
      'faq_a6':
          '如果应用运行异常，请尝试重启手机。如果你刚刚更新了应用，可能需要在安卓设置中清除应用缓存。如果所有方法都无效，你可以使用个人资料中的“重置所有进度”选项。',
      'faq_q7': '我可以跳过关卡吗？',
      'faq_a7': '可以！虽然我们建议遵循循序渐进的路径，但你完全可以根据自己当前的舒适度，自由选择地图上的任何关卡。',
      'faq_q8': '如果任务太难了怎么办？',
      'faq_a8': '虽然我们鼓励你努力完成任务，但如果确实感到困难，只需返回主屏幕并重新进入，即可刷新并更换当前任务。',
      'faq_q9': '联系我们',
      'faq_a9':
          '我们非常期待听到用户对这款应用的看法以及对未来更新的建议。我们渴望了解应用对用户的帮助程度，以及它还有哪些不足与需要改进的地方。请随时将反馈发送至 feedback.bloom@gmail.com，我们不胜感激！',
      // Task Screen
      'stage_label': '阶段: {0}',
      'keep_growing': '非常棒，{0}',
      'current_challenge': '你当前阶段的挑战任务：',
      'stage_mastered': '成功掌握该阶段！',
      'all_done': '你已完成该阶段的所有挑战。',
      'return_map': '返回地图',
      'well_done': '做得好！',
      'i_completed': '我已完成此任务',
      'level_up_suggestion_title': '升级建议',
      'level_up_suggestion_message': '你已在这个级别年完成了10项任务！你已准备好进入下一阶段。想升级吗？',
      'stay_here': '留在这里',
      'move_to_next_level': '进入下一个级别',
      // Reflection Screen
      'reflect': '复盘你的成长',
      'challenge': '挑战',
      'anxiety_q': '你感到多么焦虑？(1-10)',
      'what_happened': '实际发生了什么？',
      'write_experience_hint': '写下你的这次体验...',
      'finish': '完成复盘',
      // General / Auth
      'welcome': '欢迎来到 Bloom',
      'subtitle': '一个让你安全培养自信的空间。',
      'start': '开启我的旅程',
      'guest': '以游客身份继续',
      'hello': '你好',
      'profile': '我的资料',
      'history': '我的成长历程',
      'streak': '当前连续天数',
      'best': '历史最高连续',
      'points': '自信积分',
      //Splash Screen
      'loading': '正在加载你的花园...',
    },
    'it': {
      // Progress Screen
      'your_journey': 'Il tuo percorso',
      'keep_growing_sub':
          'Ogni piccolo passo è una vittoria. Continua a crescere!',
      'how_it_works': 'Come funziona?',
      'total_points': 'Punti totali',
      'current_streak': 'Serie attuale',
      'tasks_done': 'Sfide completate',
      'rank': 'Grado',
      'days': 'Giorni',
      'contact_us': 'Contattaci',
      'contact_email_prompt': 'Per supporto e feedback, scrivici a:',
      'close': 'Chiudi',
      // Level Map Screen
      'tap_to_view_journey': 'Tocca per vedere il tuo percorso! 🌸',
      'tap_to_start': 'Tocca per iniziare la sfida',
      'choose_level': 'Scegli la tua fase di crescita:',
      'view_journey': 'Vedi il percorso di crescita',
      'progress': 'I tuoi progressi di crescita',
      'level_seedling': 'Germoglio (Seedling)',
      'level_sprout': 'Ramoscello (Sprout)',
      'level_leaf': 'Foglia (Leaf)',
      'level_stem': 'Stelo (Stem)',
      'level_bloom': 'Fioritura (Bloom)',
      // Profile Screen
      'account': 'Account',
      'display_name': 'Nome visualizzato',
      'save_name': 'Salva nome',
      'app_theme': 'Tema dell\'app',
      'select_color': 'Seleziona il tuo colore Bloom:',
      'light_mode': 'Chiaro',
      'dark_mode': 'Scuro',
      'language': 'Lingua',
      'logout': 'Disconnetti',
      'profile_updated': 'Profilo aggiornato!',
      'pick_theme_color': 'Scegli un colore per il tema',
      'done': 'Fatto',
      'reset_all_progress': 'Resetta tutti i progressi',
      'reset_confirm_title': 'Sei sicuro?',
      'reset_confirm_message':
          'Questo cancellerà permanentemente il tuo punteggio di fiducia, la tua serie e tutta la cronologia. L\'azione non può essere annullata.',
      'cancel': 'Annulla',
      'reset_everything': 'Resetta tutto',
      'reset_success': 'Tutti i progressi sono stati resettati.',
      // Shop Screen
      'shop_title': 'Negozio Bloom',
      'your_points': 'I tuoi punti',
      'available_items': 'Oggetti disponibili',
      'streak_freeze': 'Congela Serie',
      'protects_streak': 'Protegge la tua serie dall\'azzeramento',
      'your_inventory': 'Il tuo inventario',
      'owned': 'Posseduto',
      'equipped': 'Equipaggiato',
      'equip_freeze': 'Equipaggia Congelamento',
      'buy': 'Compra',
      'freeze_purchased': 'Congelamento acquistato!',
      'not_enough_points': 'Punti insufficienti!',
      'freeze_equipped': 'Congelamento equipaggiato!',
      // Milestone Screen
      'your_growth_path': 'Il tuo cammino di crescita',
      'the_awakening': 'Il Risveglio',
      'seed_badge': 'Distintivo Seme',
      'first_spark': 'Prima Scintilla',
      'bronze_leaf': 'Foglia di Bronzo',
      'social_courage': 'Coraggio Sociale',
      'silver_sprout': 'Germoglio d\'Argent',
      'confidence_bloom': 'Fioritura di Fiducia',
      'gold_flower': 'Fiore d\'Oro',
      'mastery': 'Maestria',
      'diamond_crown': 'Corona di Diamante',
      'milestone_claimed': 'Riscattato',
      'milestone_need_score': 'Serve il {0}%',
      'milestone_claim_reward': 'Riscatta +{0} punti',
      'milestone_reward_toast': 'Riscattato! +{0} punti',
      // FAQ Screen
      'help_faq': 'Aiuto e FAQ',
      'common_questions': 'Domande frequenti',
      'keep_blooming': 'Continua a fiorire! 🌸',
      'faq_q1': 'Cos\'è Bloom?',
      'faq_a1':
          'Bloom è uno strumento di auto-aiuto progettato per aiutare le persone a ridurre l\'ansia sociale attraverso un processo chiamato \'Esposizione Graduale\'. Completando compiti sociali piccoli e gestibili, alleni il tuo cervello a capire che le interazioni sociali sono sicure e affrontabili.',
      'faq_q2': 'Come funzionano i livelli?',
      'faq_a2':
          'Iniziamo con \'Germoglio\' (compiti molto semplici) e saliamo fino a \'Fioritura\' (compiti più impegnativi). Man mano che completi le sfide, guadagni punti fiducia. Più alto è il livello, più punti ricevi!',
      'faq_q3': 'Cos\'è il Misuratore di Fiducia?',
      'faq_a3':
          'La barra di avanzamento nella schermata iniziale rappresenta la tua fiducia generale. Cresce man mano che completi i compiti. Attenzione: se smetti di esercitarti per diversi giorni, il tuo punteggio di fiducia potrebbe diminuire leggermente, ricordandoti che la fiducia è un muscolo che ha bisogno di esercizio regolare!',
      'faq_q4': 'Cos\'è una Serie (Streak)?',
      'faq_a4':
          'Una serie è il conteggio di quanti giorni consecutivi hai completato almeno un compito. La costanza è la chiave per superare l\'ansia, quindi cerca di mantenere accesa la tua fiamma!',
      'faq_q5': 'Dove sono salvati i miei dati?',
      'faq_a5':
          'La tua privacy è la nostra priorità. Tutti i tuoi progressi, la cronologia e i dati del profilo sono salvati localmente sul tuo dispositivo. Nulla viene caricato su un server cloud.',
      'faq_q6': 'Cosa faccio se l\'app si blocca?',
      'faq_a6':
          'Se l\'app si comporta in modo strano, prova a riavviare il telefono. Se hai aggiornato l\'app, potresti dover svuotare la cache dell\'app nelle impostazioni di Android. Se tutto il resto fallisce, puoi usare l\'opzione \'Resetta tutti i progressi\' nel tuo Profilo.',
      'faq_q7': 'Posso saltare i livelli?',
      'faq_a7':
          'Sì! Anche se consigliamo il percorso graduale, sei libero di scegliere qualsiasi livello dalla mappa che ritieni appropriato per il tuo attuale livello di comfort.',
      'faq_q8': 'E se il compito è troppo difficile?',
      'faq_a8':
          'Anche se ti consigliamo di provare a portare a termine il compito, puoi semplicemente tornare alla schermata iniziale e rientrare per cambiare la sfida attuale.',
      'faq_q9': 'Contattaci',
      'faq_a9':
          'Ci piacerebbe ricevere opinioni sulla nostra app dai nostri utenti e consigli per i futuri aggiornamenti. Ci piacerebbe sapere quanto l\'app funzioni bene per te, cosa le manca e cosa richiede miglioramenti. Sentiti libero di condividere un feedback su feedback.bloom@gmail.com, lo apprezzeremmo davvero molto.',
      // Task Screen
      'stage_label': 'Fase: {0}',
      'keep_growing': 'Continua così, {0}',
      'current_challenge': 'La tua sfida attuale per questa fase:',
      'stage_mastered': 'Fase Completata!',
      'all_done': 'Hai completato tutte le sfide di questa fase.',
      'return_map': 'Torna alla mappa',
      'well_done': 'Ben fatto!',
      'i_completed': 'Ho completato questa sfida',
      'level_up_suggestion_title': 'Suggerimento per salire di livello',
      'level_up_suggestion_message':
          'Hai completato 10 attività in Questo livello! Sei pronto per il livello successivo. Vuoi passare al livello successivo?',
      'stay_here': 'Resta qui',
      'move_to_next_level': 'Passa al livello successivo',
      // Reflection Screen
      'reflect': 'Rifletti sulla tua crescita',
      'challenge': 'Sfida',
      'anxiety_q': 'Quanto ti sei sentito ansioso? (1-10)',
      'what_happened': 'Cosa è successo davvero?',
      'write_experience_hint': 'Scrivi la tua esperienza...',
      'finish': 'Termina riflessione',
      // General / Auth
      'welcome': 'Benvenuto su Bloom',
      'subtitle': 'Uno spazio sicuro per far crescere la tua fiducia.',
      'start': 'Inizia il mio viaggio',
      'guest': 'Continua come ospite',
      'hello': 'Ciao',
      'profile': 'Il mio profilo',
      'history': 'Il mio percorso di crescita',
      'streak': 'Serie attuale',
      'best': 'Serie migliore',
      'points': 'Punti fiducia',
      //Splash Screen
      'loading': 'Caricamento del tuo giardino...',
    },
    'ru': {
      // Progress Screen
      'your_journey': 'Твой путь',
      'keep_growing_sub': 'Каждый маленький шаг — это победа. Продолжай расти!',
      'how_it_works': 'Как это работает?',
      'total_points': 'Всего очков',
      'current_streak': 'Текущая серия',
      'tasks_done': 'Выполнено заданий',
      'rank': 'Ранг',
      'days': 'Дней',
      'contact_us': 'Связаться с нами',
      'contact_email_prompt': 'Для поддержки и отзывов пиши нам на:',
      'close': 'Закрыть',
      // Level Map Screen
      'tap_to_view_journey': 'Нажми, чтобы увидеть свой путь! 🌸',
      'tap_to_start': 'Нажми, чтобы начать задание',
      'choose_level': 'Выбери свой этап роста:',
      'view_journey': 'Посмотреть путь роста',
      'progress': 'Твой прогресс роста',
      'level_seedling': 'Сеянец (Seedling)',
      'level_sprout': 'Росток (Sprout)',
      'level_leaf': 'Листок (Leaf)',
      'level_stem': 'Стебель (Stem)',
      'level_bloom': 'Цветение (Bloom)',
      // Profile Screen
      'account': 'Аккаунт',
      'display_name': 'Отображаемое имя',
      'save_name': 'Сохранить имя',
      'app_theme': 'Тема приложения',
      'select_color': 'Выбери свой цвет Bloom:',
      'light_mode': 'Светлая',
      'dark_mode': 'Темная',
      'language': 'Язык',
      'logout': 'Выйти',
      'profile_updated': 'Профиль обновлен!',
      'pick_theme_color': 'Выбери цвет темы',
      'done': 'Готово',
      'reset_all_progress': 'Сбросить весь прогресс',
      'reset_confirm_title': 'Ты уверен?',
      'reset_confirm_message':
          'Это навсегда удалит твои очки уверенности, серию и всю историю. Это действие нельзя отменить.',
      'cancel': 'Отмена',
      'reset_everything': 'Сбросить всё',
      'reset_success': 'Весь прогресс был сброшен.',
      // Shop Screen
      'shop_title': 'Магазин Bloom',
      'your_points': 'Твои очки',
      'available_items': 'Доступные предметы',
      'streak_freeze': 'Заморозка серии',
      'protects_streak': 'Защищает серию от сброса',
      'your_inventory': 'Твой инвентарь',
      'owned': 'Куплено',
      'equipped': 'Активно',
      'equip_freeze': 'Активировать заморозку',
      'buy': 'Купить',
      'freeze_purchased': 'Заморозка куплена!',
      'not_enough_points': 'Недостаточно очков!',
      'freeze_equipped': 'Заморозка активирована!',
      // Milestone Screen
      'your_growth_path': 'Твой путь развития',
      'the_awakening': 'Пробуждение',
      'seed_badge': 'Значок «Семечко»',
      'first_spark': 'Первая искра',
      'bronze_leaf': 'Бронзовый лист',
      'social_courage': 'Социальная смелость',
      'silver_sprout': 'Серебряный росток',
      'confidence_bloom': 'Цветение уверенности',
      'gold_flower': 'Золотой цветок',
      'mastery': 'Мастерство',
      'diamond_crown': 'Алмазная корона',
      'milestone_claimed': 'Получено',
      'milestone_need_score': 'Нужно {0}%',
      'milestone_claim_reward': 'Забрать +{0} очков',
      'milestone_reward_toast': 'Получено! +{0} очков',
      // FAQ Screen
      'help_faq': 'Помощь и FAQ',
      'common_questions': 'Частые вопросы',
      'keep_blooming': 'Продолжай цвести! 🌸',
      'faq_q1': 'Что такое Bloom?',
      'faq_a1':
          'Bloom — это инструмент самопомощи, созданный для снижения социальной тревожности с помощью метода под названием «Градуированная экспозиция». Выполняя маленькие, посильные социальные задания, ты тренируешь свой мозг понимать, что общение с людьми безопасно и контролируемо.',
      'faq_q2': 'Как работают уровни?',
      'faq_a2':
          'Мы начинаем с этапа «Сеянец» (очень простые задания) и поднимаемся до «Цветения» (более сложные вызовы). За каждое выполненное задание ты получаешь очки уверенности. Чем выше уровень, тем больше очков ты зарабатываешь!',
      'faq_q3': 'Что такое индикатор уверенности?',
      'faq_a3':
          'Шкала прогресса на главном экране показывает твой общий уровень уверенности. Она растет по мере выполнения заданий. Будь внимателен: если забросить практику на несколько дней, уровень уверенности может немного снизиться, напоминая о том, что уверенность — это мышца, требующая регулярных тренировок!',
      'faq_q4': 'Что такое серия дней (Streak)?',
      'faq_a4':
          'Серия дней — это количество дней подряд, когда ты выполнил хотя бы одно задание. Регулярность — главный ключ к преодолению тревожности, так что старайся поддерживать свой огонь!',
      'faq_q5': 'Где хранятся мои данные?',
      'faq_a5':
          'Твоя конфиденциальность — наш приоритет. Весь твой прогресс, история и данные профиля хранятся локально на твоем собственном устройстве. Ничего не загружается на облачные серверы.',
      'faq_q6': 'Что делать, если приложение вылетает или работает со сбоями?',
      'faq_a6':
          'Если приложение ведет себя странно, попробуй перезагрузить телефон. Если ты только что обновил приложение, возможно, потребуется очистить его кэш в настройках Android. Если ничего не помогает, ты можешь использовать функцию «Сбросить весь прогресс» в своем Профиле.',
      'faq_q7': 'Можно ли пропускать уровни?',
      'faq_a7':
          'Да! Хотя мы рекомендуем идти постепенно, ты можешь свободно выбирать любой уровень на карте, который соответствует твоему текущему уровню комфорта.',
      'faq_q8': 'Что делать, если задание кажется слишком сложным?',
      'faq_a8':
          'Мы советуем все же попробовать выполнить его, но если это совсем трудно, ты можешь просто вернуться на главный экран и зайти снова, чтобы сменить текущее задание.',
      'faq_q9': 'Связаться с нами',
      'faq_a9':
          'Мы будем рады услышать отзывы пользователей о нашем приложении и предложения для будущих обновлений. Нам важно знать, насколько хорошо приложение помогает людям, чего в нем не хватает и что нужно улучшить. Пожалуйста, делись своим мнением по адресу feedback.bloom@gmail.com, мы будем очень благодарны.',
      // Task Screen
      'stage_label': 'Этап: {0}',
      'keep_growing': 'Так держать, {0}',
      'current_challenge': 'Твое текущее задание на этом этапе:',
      'stage_mastered': 'Этап пройден!',
      'all_done': 'Ты завершил все задания на этом этапе.',
      'return_map': 'Вернуться на карту',
      'well_done': 'Отлично сработано!',
      'i_completed': 'Я выполнил это',
      'level_up_suggestion_title': 'Предложение повышения уровня',
      'level_up_suggestion_message':
          'Вы выполнили 10 заданий в Этот уровень! Вы готовы к следующему уровню. Хотите перейти на следующий?',
      'stay_here': 'Оставайся здесь',
      'move_to_next_level': 'Перейти на следующий уровень',
      // Reflection Screen
      'reflect': 'Обдумай свой рост',
      'challenge': 'Испытание',
      'anxiety_q': 'Насколько тревожно ты себя чувствовал? (1-10)',
      'what_happened': 'Что произошло на самом деле?',
      'write_experience_hint': 'Опиши свой опыт...',
      'finish': 'Завершить размышление',
      // General / Auth
      'welcome': 'Добро пожаловать в Bloom',
      'subtitle': 'Безопасное пространство для развития уверенности в себе.',
      'start': 'Начать мой путь',
      'guest': 'Продолжить как гость',
      'hello': 'Привет',
      'profile': 'Мой профиль',
      'history': 'Мой путь роста',
      'streak': 'Текущая серия',
      'best': 'Лучшая серия',
      'points': 'Очки уверенности',
      //Splash Screen
      'loading': 'Загрузка твоего сада...',
    },
    'tr': {
      // Progress Screen
      'your_journey': 'Senin Yolculuğun',
      'keep_growing_sub': 'Her küçük adım bir zaferdir. Büyümeye devam et!',
      'how_it_works': 'Nasıl çalışır?',
      'total_points': 'Toplam Puan',
      'current_streak': 'Mevcut Seri',
      'tasks_done': 'Tamamlanan Görevler',
      'rank': 'Derece',
      'days': 'Gün',
      'contact_us': 'Bize Ulaşın',
      'contact_email_prompt':
          'Destek ve geri bildirim için bize e-posta gönderin:',
      'close': 'Kapat',
      // Level Map Screen
      'tap_to_view_journey': 'Yolculuğunu görmek için dokun! 🌸',
      'tap_to_start': 'Mücadeleye başlamak için dokun',
      'choose_level': 'Gelişim aşamanı seç:',
      'view_journey': 'Gelişim Yolculuğunu Gör',
      'progress': 'Gelişim İlerlemen',
      'level_seedling': 'Fide (Seedling)',
      'level_sprout': 'Filiz (Sprout)',
      'level_leaf': 'Yaprak (Leaf)',
      'level_stem': 'Gövde (Stem)',
      'level_bloom': 'Çiçek Açma (Bloom)',
      // Profile Screen
      'account': 'Hesap',
      'display_name': 'Görünen Ad',
      'save_name': 'Adı Kaydet',
      'app_theme': 'Uygulama Teması',
      'select_color': 'Bloom Rengini Seç:',
      'light_mode': 'Açık',
      'dark_mode': 'Koyu',
      'language': 'Dil',
      'logout': 'Çıkış Yap',
      'profile_updated': 'Profil güncellendi!',
      'pick_theme_color': 'Bir tema rengi seç',
      'done': 'Tamam',
      'reset_all_progress': 'Tüm İlerlemeyi Sıfırla',
      'reset_confirm_title': 'Emin misin?',
      'reset_confirm_message':
          'Bu işlem güven puanını, serini ve tüm geçmişini kalıcı olarak silecektir. Bu işlem geri alınamaz.',
      'cancel': 'İptal',
      'reset_everything': 'Her Şeyi Sıfırla',
      'reset_success': 'Tüm ilerleme sıfırlandı.',
      // Shop Screen
      'shop_title': 'Bloom Mağazası',
      'your_points': 'Puanların',
      'available_items': 'Mevcut Ürünler',
      'streak_freeze': 'Seri Dondurucu',
      'protects_streak': 'Serinin sıfırlanmasını önler',
      'your_inventory': 'Envanterin',
      'owned': 'Sahip Olunan',
      'equipped': 'Kuşanıldı',
      'equip_freeze': 'Dondurucuyu Kuşan',
      'buy': 'Satın Al',
      'freeze_purchased': 'Dondurucu Satın Alındı!',
      'not_enough_points': 'Yetersiz puan!',
      'freeze_equipped': 'Dondurucu kuşanıldı!',
      // Milestone Screen
      'your_growth_path': 'Gelişim Yolun',
      'the_awakening': 'Uyanış',
      'seed_badge': 'Tohum Rozeti',
      'first_spark': 'İlk Kıvılcım',
      'bronze_leaf': 'Bronz Yaprak',
      'social_courage': 'Sosyal Cesaret',
      'silver_sprout': 'Gümüş Filiz',
      'confidence_bloom': 'Güven Çiçeği',
      'gold_flower': 'Altın Çiçek',
      'mastery': 'Ustalık',
      'diamond_crown': 'Elmas Taç',
      'milestone_claimed': 'Alındı',
      'milestone_need_score': '%{0} Gerekli',
      'milestone_claim_reward': '+{0} Puan Al',
      'milestone_reward_toast': 'Alındı! +{0} Puan',
      // FAQ Screen
      'help_faq': 'Yardım ve SSS',
      'common_questions': 'Sıkça Sorulan Sorular',
      'keep_blooming': 'Çiçek Açmaya Devam Et! 🌸',
      'faq_q1': 'Bloom nedir?',
      'faq_a1':
          'Bloom, \'Kademeli Maruz Bırakma\' adı verilen bir süreçle insanların sosyal kaygılarını azaltmalarına yardımcı olmak için tasarlanmış bir kendi kendine yardım aracıdır. Küçük, yönetilebilir sosyal görevleri tamamlayarak beynine sosyal etkileşimlerin güvenli ve başa çıkılabilir olduğunu öğretirsin.',
      'faq_q2': 'Seviyeler nasıl çalışır?',
      'faq_a2':
          '\'Fide\' (çok kolay görevler) ile başlar ve \'Çiçek Açma\' (daha zorlu görevler) seviyesine kadar yükseliriz. Görevleri tamamladıkça güven puanları kazanırsın. Seviye ne kadar yüksek olursa, o kadar çok puan kazanırsın!',
      'faq_q3': 'Güven Ölçer nedir?',
      'faq_a3':
          'Ana ekranındaki ilerleme çubuğu genel güvenini temsil eder. Görevleri tamamladıkça büyür. Dikkatli ol: Birkaç gün pratik yapmayı bırakırsan güven puanın hafifçe düşebilir; bu da sana güvenin düzenli egzersiz gerektiren bir kas olduğunu hatırlatır!',
      'faq_q4': 'Seri (Streak) nedir?',
      'faq_a4':
          'Seri, üst üste kaç gün boyunca en az bir görevi tamamladığının sayısıdır. İstikrar, kaygıyı yenmenin anahtarıdır, bu yüzden alevini canlı tutmaya çalış!',
      'faq_q5': 'Verilerim nerede saklanıyor?',
      'faq_a5':
          'Gizliliğin bizim önceliğimizdir. Tüm ilerlemen, geçmişin ve profil verilerin yerel olarak kendi cihazında saklanır. Bulut sunucusuna hiçbir şey yüklenmez.',
      'faq_q6': 'Uygulama çökerse ne yapmalıyım?',
      'faq_a6':
          'Uygulama tuhaf davranıyorsa telefonunu yeniden başlatmayı dene. Uygulamayı güncellediysen, Android ayarlarından uygulama önbelleğini temizlemen gerekebilir. Her şey başarısız olursa, Profilindeki \'Tüm İlerlemeyi Sıfırla\' seçeneğini kullanabilirsin.',
      'faq_q7': 'Seviyeleri atlayabilir miyim?',
      'faq_a7':
          'Evet! Kademeli yolu önersek de, haritadan mevcut konfor seviyene uygun hissettiğin herhangi bir seviyeyi seçmekte özgürsün.',
      'faq_q8': 'Ya görev çok zorsa?',
      'faq_a8':
          'Görevi başarmaya çalışmanı önersek de, sadece ana ekrana geri dönebilir ve mevcut görevi değiştirmek için tekrar giriş yapabilirsin.',
      'faq_q9': 'Bize Ulaşın',
      'faq_a9':
          'Kullanıcılarımızdan uygulamamız hakkındaki görüşlerini ve gelecekteki güncellemeler için önerilerini duymayı çok isteriz. Uygulamanın kullanıcılar için ne kadar iyi çalıştığını, neyin eksik olduğunu ve neyin geliştirilmesi gerektiğini bilmek isteriz. Lütfen feedback.bloom@gmail.com adresinden geri bildirim paylaşmaktan çekinme, gerçekten minnettar oluruz.',
      // Task Screen
      'stage_label': 'Aşama: {0}',
      'keep_growing': 'Büyümeye devam et, {0}',
      'current_challenge': 'Bu aşama için mevcut mücadelen:',
      'stage_mastered': 'Aşama Tamamlandı!',
      'all_done': 'Bu aşamadaki tüm mücadeleleri tamamladın.',
      'return_map': 'Haritaya Geri Dön',
      'well_done': 'Tebrikler!',
      'i_completed': 'Bunu Tamamladım',
      'level_up_suggestion_title': 'Seviye Atlama Önerisi',
      'level_up_suggestion_message':
          'Bu seviye sürede 10 görevi tamamladınız! Bir sonraki seviyeye geçmeye hazırsınız. Bir üst seviyeye geçmek ister misiniz?',
      'stay_here': 'Burada kal',
      'move_to_next_level': 'Sonraki Seviyeye Geçin',
      // Reflection Screen
      'reflect': 'Gelişimini değerlendir',
      'challenge': 'Meydan okumak',
      'anxiety_q': 'Ne kadar kaygılı hissettin? (1-10)',
      'what_happened': 'Gerçekte ne oldu?',
      'write_experience_hint': 'Deneyimin hakkında yaz...',
      'finish': 'Değerlendirmeyi Bitir',
      // General / Auth
      'welcome': 'Bloom\'a Hoş Geldin',
      'subtitle': 'Özgüvenini geliştirebileceğin güvenli bir alan.',
      'start': 'Yolculuğumu Başlat',
      'guest': 'Misafir olarak Devam Et',
      'hello': 'Merhaba',
      'profile': 'Profilim',
      'history': 'Gelişim Yolculuğum',
      'streak': 'Mevcut Seri',
      'best': 'En İyi Seri',
      'points': 'Güven Puanı',
      //Splash Screen
      'loading': 'Bahçen yükleniyor...',
    },
    'nl': {
      // Progress Screen
      'your_journey': 'Jouw Reis',
      'keep_growing_sub': 'Elke kleine stap is een overwinning. Blijf groeien!',
      'how_it_works': 'Hoe werkt het?',
      'total_points': 'Totale Punten',
      'current_streak': 'Huidige Reeks',
      'tasks_done': 'Taken Voltooid',
      'rank': 'Rang',
      'days': 'Dagen',
      'contact_us': 'Contact',
      'contact_email_prompt':
          'Voor ondersteuning en feedback kun je mailen naar:',
      'close': 'Sluiten',
      // Level Map Screen
      'tap_to_view_journey': 'Tik om je reis te bekijken! 🌸',
      'tap_to_start': 'Tik om de uitdaging te starten',
      'choose_level': 'Kies je groeifase:',
      'view_journey': 'Bekijk Groeireis',
      'progress': 'Jouw Groeivoortgang',
      'level_seedling': 'Zaailing (Seedling)',
      'level_sprout': 'Spruit (Sprout)',
      'level_leaf': 'Blad (Leaf)',
      'level_stem': 'Stengel (Stem)',
      'level_bloom': 'Bloei (Bloom)',
      // Profile Screen
      'account': 'Account',
      'display_name': 'Schermnaam',
      'save_name': 'Naam Opslaan',
      'app_theme': 'App-thema',
      'select_color': 'Kies je Bloom-kleur:',
      'light_mode': 'Licht',
      'dark_mode': 'Donker',
      'language': 'Taal',
      'logout': 'Uitloggen',
      'profile_updated': 'Profiel bijgewerkt!',
      'pick_theme_color': 'Kies een themakleur',
      'done': 'Klaar',
      'reset_all_progress': 'Reset Alle Voortgang',
      'reset_confirm_title': 'Weet je het zeker?',
      'reset_confirm_message':
          'Dit zal je zelfvertrouwenscore, reeks en volledige geschiedenis permanent verwijderen. Dit kan niet ongedaan worden gemaakt.',
      'cancel': 'Annuleren',
      'reset_everything': 'Alles Resetten',
      'reset_success': 'Alle voortgang is gereset.',
      // Shop Screen
      'shop_title': 'Bloom Winkel',
      'your_points': 'Jouw Punten',
      'available_items': 'Beschikbare Items',
      'streak_freeze': 'Reeks Bevriezen',
      'protects_streak': 'Beschermt je reeks tegen een reset',
      'your_inventory': 'Jouw Inventaris',
      'owned': 'In bezit',
      'equipped': 'Uitgerust',
      'equip_freeze': 'Bevriezing Uitrusten',
      'buy': 'Kopen',
      'freeze_purchased': 'Reeks Bevriezen Gekocht!',
      'not_enough_points': 'Niet genoeg punten!',
      'freeze_equipped': 'Bevriezing uitgerust!',
      // Milestone Screen
      'your_growth_path': 'Jouw Groeipad',
      'the_awakening': 'Het Ontwaken',
      'seed_badge': 'Zaadje Badge',
      'first_spark': 'Eerste Vonk',
      'bronze_leaf': 'Bronzen Blad',
      'social_courage': 'Sociale Moed',
      'silver_sprout': 'Zilveren Spruit',
      'confidence_bloom': 'Zelfvertrouwen Bloei',
      'gold_flower': 'Gouden Bloem',
      'mastery': 'Meesterschap',
      'diamond_crown': 'Diamanten Kroon',
      'milestone_claimed': 'Geclaimd',
      'milestone_need_score': '{0}% Nodig',
      'milestone_claim_reward': 'Claim +{0} ptn',
      'milestone_reward_toast': 'Geclaimd! +{0} ptn',
      // FAQ Screen
      'help_faq': 'Hulp & FAQ',
      'common_questions': 'Veelgestelde Vragen',
      'keep_blooming': 'Blijf Bloeien! 🌸',
      'faq_q1': 'Wat is Bloom?',
      'faq_a1':
          'Bloom is een zelfhulptool die is ontworpen om mensen te helpen sociale angst verminderen via een proces genaamd \'Geleidelijke Blootstelling\'. Door kleine, behapbare sociale taken te voltooien, train je je hersenen om te beseffen dat sociale interacties veilig en beheersbaar zijn.',
      'faq_q2': 'Hoe werken de niveaus?',
      'faq_a2':
          'We beginnen met \'Zaailing\' (zeer makkelijke taken) en stromen door naar \'Bloei\' (uitdagendere taken). Naarmate je taken voltooit, verdien je zelfvertrouwenpunten. Hoe hoger het niveau, hoe meer punten je verdient!',
      'faq_q3': 'Wat is de Zelfvertrouwenmeter?',
      'faq_a3':
          'De voortgangsbalk op je startscherm vertegenwoordigt je algehele zelfvertrouwen. Deze groeit naarmate je taken voltooit. Pas op: als je een aantal dagen stopt met oefenen, kan je zelfvertrouwenscore een beetje dalen om je eraan te herinneren dat zelfvertrouwen een spier is die regelmatige training nodig heeft!',
      'faq_q4': 'Wat is een Reeks (Streak)?',
      'faq_a4':
          'Een reeks is de telling van hoeveel opeenvolgende dagen je minstens één taak hebt voltooid. Consistentie is de sleutel tot het overwinnen van angst, dus probeer je vlam brandend te houden!',
      'faq_q5': 'Waar worden mijn gegevens opgeslagen?',
      'faq_a5':
          'Jouw privacy is onze prioriteit. Al je voortgang, geschiedenis en profielgegevens worden lokaal op je eigen apparaat opgeslagen. Er wordt niets geüpload naar een cloudserver.',
      'faq_q6': 'Wat moet ik doen als de app vastloopt?',
      'faq_a6':
          'Als de app zich vreemd gedraagt, probeer dan je telefoon opnieuw op te starten. Als je de app hebt bijgewerkt, moet je mogelijk de app-cache wissen in je Android-instellingen. Als al het andere faalt, kun je de optie \'Reset Alle Voortgang\' in je Profiel gebruiken.',
      'faq_q7': 'Kan ik niveaus overslaan?',
      'faq_a7':
          'Ja! Hoewel we het geleidelijke pad aanbevelen, ben je vrij om elk niveau op de kaart te kiezen dat past bij je huidige comfortniveau.',
      'faq_q8': 'Wat als de taak heel erg moeilijk is?',
      'faq_a8':
          'Hoewel we je aanraden om te proberen de taak te voltooien, kun je ook gewoon teruggaan naar het startscherm en opnieuw openen om de huidige taak te veranderen.',
      'faq_q9': 'Contact',
      'faq_a9':
          'We horen graag van onze gebruikers wat ze van de app vinden en ontvangen graag aanbevelingen voor toekomstige updates. We willen graag horen hoe goed de app werkt voor gebruikers, en wat er nog mist of verbeterd moet worden. Deel gerust je feedback via feedback.bloom@gmail.com, dat zouden we enorm waarderen.',
      // Task Screen
      'stage_label': 'Fase: {0}',
      'keep_growing': 'Blijf groeien, {0}',
      'current_challenge': 'Je huidige uitdaging voor deze fase:',
      'stage_mastered': 'Fase Voltooid!',
      'all_done': 'Je hebt alle uitdagingen in deze fase voltooid.',
      'return_map': 'Terug naar Kaart',
      'well_done': 'Goed Gedaan!',
      'i_completed': 'Ik Heb Dit Voltooid',
      'level_up_suggestion_title': 'Suggestie voor een hoger niveau',
      'level_up_suggestion_message':
          'Je hebt 10 taken voltooid in Dit niveau! Je bent klaar voor het volgende niveau. Wil je door naar het volgende niveau?',
      'stay_here': 'Blijf hier',
      'move_to_next_level': 'Ga naar het volgende niveau',
      // Reflection Screen
      'reflect': 'Reflecteer op je groei',
      'challenge': 'Uitdaging',
      'anxiety_q': 'Hoe angstig voelde je je? (1-10)',
      'what_happened': 'Wat gebeurde er werkelijk?',
      'write_experience_hint': 'Schrijf over je ervaring...',
      'finish': 'Reflectie Voltooien',
      // General / Auth
      'welcome': 'Welkom bij Bloom',
      'subtitle': 'Een veilige plek om je zelfvertrouwen te laten groeien.',
      'start': 'Start Mijn Reis',
      'guest': 'Doorgaan als Gast',
      'hello': 'Hallo',
      'profile': 'Mijn Profiel',
      'history': 'Mijn Groeireis',
      'streak': 'Huidige Reeks',
      'best': 'Beste Reeks',
      'points': 'Zelfvertrouwen Punten',
      //Splash Screen
      'loading': 'Je tuin laden...',
    },
    'bn': {
      // Progress Screen
      'your_journey': 'আপনার যাত্রা',
      'keep_growing_sub': 'প্রতিটি ছোট পদক্ষেপই একটি জয়। এগিয়ে যান!',
      'how_it_works': 'এটি কীভাবে কাজ করে?',
      'total_points': 'মোট পয়েন্ট',
      'current_streak': 'বর্তমান ধারাবাহিকতা',
      'tasks_done': 'সম্পন্ন কাজ',
      'rank': 'র‌্যাংক',
      'days': 'দিন',
      'contact_us': 'যোগাযোগ করুন',
      'contact_email_prompt': 'সহায়তা এবং মতামতের জন্য আমাদের ইমেল করুন:',
      'close': 'বন্ধ করুন',
      // Level Map Screen
      'tap_to_view_journey': 'আপনার যাত্রা দেখতে আলতো চাপুন! 🌸',
      'tap_to_start': 'চ্যালেঞ্জ শুরু করতে আলতো চাপুন',
      'choose_level': 'আপনার বৃদ্ধির স্তরটি বেছে নিন:',
      'view_journey': 'বৃদ্ধির যাত্রা দেখুন',
      'progress': 'আপনার অগ্রগতির হার',
      'level_seedling': 'চারাগাছ (Seedling)',
      'level_sprout': 'অঙ্কুর (Sprout)',
      'level_leaf': 'কচি পাতা (Leaf)',
      'level_stem': 'কাণ্ড (Stem)',
      'level_bloom': 'পূর্ণ বিকাশ (Bloom)',
      // Profile Screen
      'account': 'অ্যাকাউন্ট',
      'display_name': 'প্রদর্শিত নাম',
      'save_name': 'নাম সংরক্ষণ করুন',
      'app_theme': 'অ্যাপ থিম',
      'select_color': 'আপনার ব্লুম রঙটি বেছে নিন:',
      'light_mode': 'লাইট',
      'dark_mode': 'ডার্ক',
      'language': 'ভাষা',
      'logout': 'লগআউট',
      'profile_updated': 'প্রোফাইল আপডেট করা হয়েছে!',
      'pick_theme_color': 'থিমের রঙ বেছে নিন',
      'done': 'সম্পন্ন',
      'reset_all_progress': 'সব অগ্রগতি রিসেট করুন',
      'reset_confirm_title': 'আপনি কি নিশ্চিত?',
      'reset_confirm_message':
          'এটি আপনার আত্মবিশ্বাসের স্কোর, ধারাবাহিকতা এবং সমস্ত ইতিহাস স্থায়ীভাবে মুছে ফেলবে। এটি আর ফিরিয়ে আনা যাবে না।',
      'cancel': 'বাতিল',
      'reset_everything': 'সব কিছু রিসেট করুন',
      'reset_success': 'সমস্ত অগ্রগতি রিসেট করা হয়েছে।',
      // Shop Screen
      'shop_title': 'ব্লুম শপ',
      'your_points': 'আপনার পয়েন্ট',
      'available_items': 'উপলব্ধ আইটেম',
      'streak_freeze': 'ধারাবাহিকতা ফ্রিজ',
      'protects_streak': 'ধারাবাহিকতা রিসেট হওয়া থেকে রক্ষা করে',
      'your_inventory': 'আপনার ইনভেন্টরি',
      'owned': 'কেনা হয়েছে',
      'equipped': 'ব্যবহৃত',
      'equip_freeze': 'ফ্রিজ ব্যবহার করুন',
      'buy': 'কিনুন',
      'freeze_purchased': 'ফ্রিজ কেনা হয়েছে!',
      'not_enough_points': 'পর্যাপ্ত পয়েন্ট নেই!',
      'freeze_equipped': 'ফ্রিজ সক্রিয় করা হয়েছে!',
      // Milestone Screen
      'your_growth_path': 'আপনার বৃদ্ধির পথ',
      'the_awakening': 'জাগরণ',
      'seed_badge': 'বীজ ব্যাজ',
      'first_spark': 'প্রথম স্ফুলিঙ্গ',
      'bronze_leaf': 'ব্রোঞ্জ পাতা',
      'social_courage': 'সামাজিক সাহস',
      'silver_sprout': 'রুপালি অঙ্কুর',
      'confidence_bloom': 'আত্মবিশ্বাসের বিকাশ',
      'gold_flower': 'স্বর্ণালী ফুল',
      'mastery': 'দক্ষতা',
      'diamond_crown': 'হীরক মুকুট',
      'milestone_claimed': 'সংগৃহীত',
      'milestone_need_score': '{0}% প্রয়োজন',
      'milestone_claim_reward': 'পুরস্কার নিন +{0} পয়েন্ট',
      'milestone_reward_toast': 'পুরস্কার নেওয়া হয়েছে! +{0} পয়েন্ট',
      // FAQ Screen
      'help_faq': 'সহায়তা এবং সাধারণ জিজ্ঞাসা',
      'common_questions': 'সাধারণ জিজ্ঞাসা',
      'keep_blooming': 'বিকাশ ছড়াতে থাকুন! 🌸',
      'faq_q1': 'ব্লুম (Bloom) কী?',
      'faq_a1':
          'ব্লুম হলো একটি সেলফ-হেল্প টুল যা \'গ্র্যাডেড এক্সপোজার\' নামক একটি প্রক্রিয়ার মাধ্যমে মানুষের সামাজিক উদ্বেগ বা সংকোচ কমাতে সাহায্য করার জন্য ডিজাইন করা হয়েছে। ছোট ছোট, সহজে করা যায় এমন সামাজিক কাজ সম্পন্ন করার মাধ্যমে আপনি আপনার মস্তিষ্ককে এটি বুঝতে সাহায্য করেন যে সামাজিক যোগাযোগগুলো নিরাপদ এবং স্বাভাবিক।',
      'faq_q2': 'স্তর বা লেভেলগুলো কীভাবে কাজ করে?',
      'faq_a2':
          'আমরা \'চারাগাছ\' (খুব সহজ কাজ) দিয়ে শুরু করি এবং ধীরে ধীরে \'পূর্ণ বিকাশ\' (অপেক্ষাকৃত কঠিন কাজ)-এর দিকে এগিয়ে যাই। কাজগুলো সম্পন্ন করার সাথে সাথে আপনি আত্মবিশ্বাসের পয়েন্ট অর্জন করবেন। লেভেল যত বেশি হবে, আপনি তত বেশি পয়েন্ট পাবেন!',
      'faq_q3': 'কনফিডেন্স মিটার কী?',
      'faq_a3':
          'আপনার হোম স্ক্রিনের প্রোগ্রেস বারটি আপনার সামগ্রিক আত্মবিশ্বাসকে নির্দেশ করে। কাজ সম্পন্ন করার সাথে সাথে এটি বৃদ্ধি পায়। তবে সাবধান: আপনি যদি বেশ কয়েক দিন অনুশীলন করা বন্ধ করে দেন, তবে আপনার আত্মবিশ্বাসের স্কোর কিছুটা কমে যেতে পারে, যা আপনাকে মনে করিয়ে দেয় যে আত্মবিশ্বাস হলো একটি পেশীর মতো যার জন্য নিয়মিত ব্যায়াম বা অনুশীলনের প্রয়োজন!',
      'faq_q4': 'ধারাবাহিকতা বা স্ট্রিক (Streak) কী?',
      'faq_a4':
          'ধারাবাহিকতা হলো আপনি টানা কতদিন অন্তত একটি করে কাজ সম্পন্ন করেছেন তার হিসাব। উদ্বেগ কাটিয়ে ওঠার মূল চাবিকাঠি হলো ধারাবাহিকতা, তাই আপনার মনের ভেতরের এই আগুনকে জ্বালিয়ে রাখার চেষ্টা করুন!',
      'faq_q5': 'আমার ডেটা কোথায় সংরক্ষিত থাকে?',
      'faq_a5':
          'আপনার গোপনীয়তা আমাদের সর্বোচ্চ অগ্রাধিকার। আপনার সমস্ত অগ্রগতি, ইতিহাস এবং প্রোফাইল ডেটা আপনার নিজস্ব ডিভাইসেই স্থানীয়ভাবে সংরক্ষিত থাকে। কোনো তথ্যই ক্লাউড সার্ভারে আপলোড করা হয় না।',
      'faq_q6': 'অ্যাপটি ক্র্যাশ করলে আমি কী করব?',
      'faq_a6':
          'অ্যাপটি যদি অদ্ভুত আচরণ করে, তবে আপনার ফোনটি রিস্টার্ট করার চেষ্টা করুন। আপনি যদি অ্যাপটি আপডেট করে থাকেন, তবে আপনার অ্যান্ড্রয়েড সেটিংস থেকে অ্যাপের ক্যাশে (Cache) পরিষ্কার করার প্রয়োজন হতে পারে। যদি কোনো কিছুতেই কাজ না হয়, তবে আপনি আপনার প্রোফাইল থেকে \'সব অগ্রগতি রিসেট করুন\' বিকল্পটি ব্যবহার করতে পারেন।',
      'faq_q7': 'আমি কি লেভেল স্কিপ বা বাদ দিতে পারি?',
      'faq_a7':
          'হ্যাঁ! যদিও আমরা ধাপে ধাপে যাওয়ার পরামর্শ দিই, তবে আপনি ম্যাপ থেকে যেকোনো লেভেল বেছে নিতে সম্পূর্ণ স্বাধীন যা আপনার বর্তমান স্বাচ্ছন্দ্যের স্তরের সাথে মানানসই মনে হয়।',
      'faq_q8': 'কাজটি যদি খুব কঠিন মনে হয় তবে কী হবে?',
      'faq_a8':
          'যদিও আমরা আপনাকে কাজটি সম্পন্ন করার চেষ্টা করার পরামর্শ দিই, তবে আপনি কেবল হোম স্ক্রিনে ফিরে যেতে পারেন এবং বর্তমান কাজটি পরিবর্তন করতে আবার প্রবেশ করতে পারেন।',
      'faq_q9': 'যোগাযোগ করুন',
      'faq_a9':
          'আমরা আমাদের ব্যবহারকারীদের কাছ থেকে অ্যাপ সম্পর্কে জানতে এবং ভবিষ্যতের আপডেটের জন্য তাদের সুপারিশ পেতে পছন্দ করব। অ্যাপটি ব্যবহারকারীদের জন্য কতটা ভালো কাজ করছে এবং এতে কীসের অভাব রয়েছে বা কোথায় উন্নতি করা দরকার তা আমাদের জানান। অনুগ্রহ করে feedback.bloom@gmail.com-এ আপনার মতামত জানাতে দ্বিধা করবেন না, আমরা এটি অত্যন্ত গুরুত্বের সাথে মূল্যায়ন করব।',
      // Task Screen
      'stage_label': 'ধাপ: {0}',
      'keep_growing': 'এগিয়ে যান, {0}',
      'current_challenge': 'এই ধাপের জন্য আপনার বর্তমান চ্যালেঞ্জ:',
      'stage_mastered': 'ধাপটি সফলভাবে সম্পন্ন হয়েছে!',
      'all_done': 'আপনি এই ধাপের সমস্ত চ্যালেঞ্জ সম্পন্ন করেছেন।',
      'return_map': 'ম্যাপে ফিরে যান',
      'well_done': 'দারুণ হয়েছে!',
      'i_completed': 'আমি এটি সম্পন্ন করেছি',
      'level_up_suggestion_title': 'লেভেল আপ সাজেশন',
      'level_up_suggestion_message':
          'আপনি এই স্তর মিনিটে 10টি কাজ সম্পন্ন করেছেন! আপনি পরবর্তী স্তরের জন্য প্রস্তুত। উপরে উঠতে চান?',
      'stay_here': 'এখানে থাকুন',
      'move_to_next_level': 'পরবর্তী স্তরে যান',
      // Reflection Screen
      'reflect': 'আপনার বৃদ্ধির ওপর আলোকপাত করুন',
      'challenge': 'চ্যালেঞ্জ',
      'anxiety_q': 'আপনি কতটা উদ্বিগ্ন বোধ করেছিলেন? (১-১০)',
      'what_happened': 'আসলে কী ঘটেছিল?',
      'write_experience_hint': 'আপনার অভিজ্ঞতা সম্পর্কে লিখুন...',
      'finish': 'আলোকপাত সম্পন্ন করুন',
      // General / Auth
      'welcome': 'ব্লুম-এ স্বাগতম',
      'subtitle': 'আপনার আত্মবিশ্বাস বাড়ানোর একটি নিরাপদ স্থান।',
      'start': 'আমার যাত্রা শুরু করুন',
      'guest': 'গেস্ট হিসেবে চালিয়ে যান',
      'hello': 'হ্যালো',
      'profile': 'আমার প্রোফাইল',
      'history': 'আমার বৃদ্ধির যাত্রা',
      'streak': 'বর্তমান ধারাবাহিকতা',
      'best': 'সেরা ধারাবাহিকতা',
      'points': 'আত্মবিশ্বাসের পয়েন্ট',
      //Splash Screen
      'loading': 'আপনার বাগান লোড হচ্ছে...',
    },
    'ta': {
      // Progress Screen
      'your_journey': 'உங்களின் பயணம்',
      'keep_growing_sub':
          'ஒவ்வொரு சிறிய அடியும் ஒரு வெற்றிதான். தொடர்ந்து வளருங்கள்!',
      'how_it_works': 'இது எப்படி வேலை செய்கிறது?',
      'total_points': 'மொத்த புள்ளிகள்',
      'current_streak': 'தற்போதைய தொடர் நாட்கள்',
      'tasks_done': 'முடித்த சவால்கள்',
      'rank': 'நிலை (Rank)',
      'days': 'நாட்கள்',
      'contact_us': 'எங்களைத் தொடர்பு கொள்ளவும்',
      'contact_email_prompt':
          'ஆதரவு மற்றும் கருத்துகளுக்கு, எங்களுக்கு மின்னஞ்சல் அனுப்பவும்:',
      'close': 'மூடு',
      // Level Map Screen
      'tap_to_view_journey': 'உங்கள் பயணத்தைக் காண தட்டவும்! 🌸',
      'tap_to_start': 'சவாலைத் தொடங்க தட்டவும்',
      'choose_level': 'உங்கள் வளர்ச்சி நிலையைத் தேர்ந்தெடுக்கவும்:',
      'view_journey': 'வளர்ச்சிப் பயணத்தைக் காண்க',
      'progress': 'உங்களின் வளர்ச்சி முன்னேற்றம்',
      'level_seedling': 'நாற்று (Seedling)',
      'level_sprout': 'முளை (Sprout)',
      'level_leaf': 'இலை (Leaf)',
      'level_stem': 'தண்டு (Stem)',
      'level_bloom': 'மலர்ச்சி (Bloom)',
      // Profile Screen
      'account': 'கணக்கு',
      'display_name': 'காண்பிக்கும் பெயர்',
      'save_name': 'பெயரைச் சேமி',
      'app_theme': 'செயலி தீம் (Theme)',
      'select_color': 'உங்கள் புளூம் நிறத்தைத் தேர்ந்தெடுக்கவும்:',
      'light_mode': 'லைட் மோட்',
      'dark_mode': 'டார்க் மோட்',
      'language': 'மொழி',
      'logout': 'வெளியேறு',
      'profile_updated': 'சுயவிவரம் புதுப்பிக்கப்பட்டது!',
      'pick_theme_color': 'தீம் நிறத்தைத் தேர்ந்தெடுக்கவும்',
      'done': 'முடிந்தது',
      'reset_all_progress': 'அனைத்து முன்னேற்றத்தையும் மீட்டமை (Reset)',
      'reset_confirm_title': 'நீங்கள் உறுதியாக இருக்கிறீர்களா?',
      'reset_confirm_message':
          'இது உங்கள் தன்னம்பிக்கை மதிப்பெண், தொடர் நாட்கள் மற்றும் அனைத்து வரலாற்றையும் நிரந்தரமாக அழித்துவிடும். இதை மாற்றியமைக்க முடியாது.',
      'cancel': 'ரத்து செய்',
      'reset_everything': 'அனைத்தையும் மீட்டமை',
      'reset_success': 'அனைத்து முன்னேற்றங்களும் மீட்டமைக்கப்பட்டன.',
      // Shop Screen
      'shop_title': 'புளூம் ஷாப்',
      'your_points': 'உங்கள் புள்ளிகள்',
      'available_items': 'கிடைக்கும் பொருட்கள்',
      'streak_freeze': 'ஸ்ட்ரீக் ஃப்ரீஸ் (Streak Freeze)',
      'protects_streak': 'தொடர் நாட்கள் பூஜ்ஜியமாவதைத் தடுக்கிறது',
      'your_inventory': 'உங்கள் பொருட்கள்',
      'owned': 'வாங்கப்பட்டது',
      'equipped': 'பயன்பாட்டில் உள்ளது',
      'equip_freeze': 'ஃப்ரீஸைப் பயன்படுத்து',
      'buy': 'வாங்கு',
      'freeze_purchased': 'ஸ்ட்ரீக் ஃப்ரீஸ் வாங்கப்பட்டது!',
      'not_enough_points': 'போதிய புள்ளிகள் இல்லை!',
      'freeze_equipped': 'ஃப்ரீஸ் செயல்படுத்தப்பட்டது!',
      // Milestone Screen
      'your_growth_path': 'உங்கள் வளர்ச்சிப் பாதை',
      'the_awakening': 'விழிப்புணர்வு',
      'seed_badge': 'விதை பேட்ஜ்',
      'first_spark': 'முதல் பொறி',
      'bronze_leaf': 'வெண்கல இலை',
      'social_courage': 'சமூகத் துணிச்சல்',
      'silver_sprout': 'வெள்ளி முளை',
      'confidence_bloom': 'தன்னம்பிக்கை மலர்ச்சி',
      'gold_flower': 'தங்க மலர்',
      'mastery': 'ஆளுமை',
      'diamond_crown': 'வைர கிரீடம்',
      'milestone_claimed': 'பெறப்பட்டது',
      'milestone_need_score': '{0}% தேவை',
      'milestone_claim_reward': 'பரிசைப் பெறு +{0} புள்ளிகள்',
      'milestone_reward_toast': 'பெறப்பட்டது! +{0} புள்ளிகள்',
      // FAQ Screen
      'help_faq': 'உதவி மற்றும் அடிக்கடி கேட்கப்படும் கேள்விகள்',
      'common_questions': 'பொதுவான கேள்விகள்',
      'keep_blooming': 'தொடர்ந்து மலருங்கள்! 🌸',
      'faq_q1': 'புளூம் (Bloom) என்றால் என்ன?',
      'faq_a1':
          'புளூம் என்பது \'கிரேடட் எக்ஸ்போஷர்\' (Graded Exposure) எனப்படும் செயல்முறையின் மூலம் மக்கள் சமூக பதற்றத்தைக் குறைக்க உதவும் வகையில் வடிவமைக்கப்பட்ட ஒரு சுய உதவி கருவியாகும். சிறிய, எளிமையான சமூக சவால்களை முடிப்பதன் மூலம், சமூக தொடர்புகள் பாதுகாப்பானவை மற்றும் கையாளக்கூடியவை என்பதை உங்கள் மூளைக்கு நீங்கள் பழக்குகிறீர்கள்.',
      'faq_q2': 'நிலைகள் (Levels) எவ்வாறு செயல்படுகின்றன?',
      'faq_a2':
          'நாம் \'நாற்று\' (மிகவும் எளிதான பணிகள்) என்பதில் தொடங்கி, \'மலர்ச்சி\' (அதிக சவாலான பணிகள்) வரை முன்னேறுகிறோம். நீங்கள் பணிகளை முடிக்கும்போது, தன்னம்பிக்கை புள்ளிகளைப் பெறுவீர்கள். நிலை அதிகமாகும் போது, அதிக புள்ளிகளைப் பெறலாம்!',
      'faq_q3': 'தன்னம்பிக்கை மீட்டர் என்றால் என்ன?',
      'faq_a3':
          'உங்கள் முகப்புத் திரையில் உள்ள முன்னேற்றப் பட்டி உங்களின் ஒட்டுமொத்த தன்னம்பிக்கையைக் குறிக்கிறது. நீங்கள் பணிகளை முடிக்கும்போது அது வளர்கிறது. கவனமாக இருங்கள்: நீங்கள் பல நாட்கள் பயிற்சி செய்வதை நிறுத்தினால், உங்கள் தன்னம்பிக்கை மதிப்பெண் சற்று குறையக்கூடும், தன்னம்பிக்கை என்பது வழக்கமான பயிற்சி தேவைப்படும் ஒரு தசை என்பதை இது நினைவூட்டுகிறது!',
      'faq_q4': 'தொடர் நாட்கள் (Streak) என்றால் என்ன?',
      'faq_a4':
          'தொடர் நாட்கள் என்பது நீங்கள் தொடர்ந்து எத்தனை நாட்கள் குறைந்தது ஒரு பணியையாவது முடித்துள்ளீர்கள் என்ற கணக்காகும். பதற்றத்தை வெல்வதற்குத் தொடர்ச்சிதான் முக்கியம், எனவே உங்கள் ஆர்வ நெருப்பைத் தொடர்ந்து எரிய வைக்க முயலுங்கள்!',
      'faq_q5': 'எனது தரவு எங்கே சேமிக்கப்படுகிறது?',
      'faq_a5':
          'உங்கள் தனியுரிமையே எங்களின் முன்னுரிமை. உங்களின் அனைத்து முன்னேற்றங்கள், வரலாறு மற்றும் சுயவிவரத் தரவுகள் உங்கள் சொந்த சாதனத்திலேயே உள்நாட்டில் (Locally) சேமிக்கப்படும். மேகக்கணி (Cloud) சேவையகத்தில் எதுவும் பதிவேற்றப்படாது.',
      'faq_q6': 'செயலி செயலிழந்தால் (Crash) நான் என்ன செய்ய வேண்டும்?',
      'faq_a6':
          'செயலி விசித்திரமாக நடந்து கொண்டால், உங்கள் தொலைபேசியை மறுதொடக்கம் (Restart) செய்ய முயற்சிக்கவும். நீங்கள் செயலியைப் புதுப்பித்திருந்தால், உங்கள் ஆண்ட்ராய்டு அமைப்புகளில் செயலி கேச் (Cache) நினைவகத்தை அழிக்க வேண்டியிருக்கலாம். எதுவும் வேலை செய்யவில்லை என்றால், உங்கள் சுயவிவரத்தில் உள்ள \'அனைத்து முன்னேற்றத்தையும் மீட்டமை\' என்ற விருப்பத்தைப் பயன்படுத்தலாம்.',
      'faq_q7': 'நான் நிலைகளைத் தவிர்க்கலாமா (Skip)?',
      'faq_a7':
          'ஆம்! படிப்படியான பாதையை நாங்கள் பரிந்துரைத்தாலும், உங்களின் தற்போதைய வசதிக்கேற்ப வரைபடத்திலிருந்து எந்தவொரு நிலையையும் தேர்ந்தெடுக்க உங்களுக்கு முழு சுதந்திரம் உள்ளது.',
      'faq_q8': 'பணி மிகவும் கடினமாக இருந்தால் என்ன செய்வது?',
      'faq_a8':
          'பணியை முடிக்க முயற்சி செய்யுமாறு நாங்கள் பரிந்துரைத்தாலும், தற்போதைய பணியை மாற்ற நீங்கள் முகப்புத் திரைக்குச் சென்று மீண்டும் உள்ளே நுழையலாம்.',
      'faq_q9': 'எங்களைத் தொடர்பு கொள்ளவும்',
      'faq_a9':
          'எங்கள் பயனர்களிடமிருந்து செயலியைப் பற்றிய கருத்துகளையும், எதிர்கால புதுப்பிப்புகளுக்கான பரிந்துரைகளையும் கேட்க நாங்கள் விரும்புகிறோம். செயலி பயனர்களுக்கு எவ்வளவு நன்றாக வேலை செய்கிறது, அதில் என்ன குறைபாடுகள் உள்ளன மற்றும் எதில் முன்னேற்றம் தேவை என்பதை அறிய விரும்புகிறோம். feedback.bloom@gmail.com இல் உங்கள் கருத்துக்களைப் பகிர்ந்து கொள்ள தயங்க வேண்டாம், அதை நாங்கள் பெரிதும் மதிப்போம்.',
      // Task Screen
      'stage_label': 'கட்டம்: {0}',
      'keep_growing': 'தொடர்ந்து வளருங்கள், {0}',
      'current_challenge': 'இந்தக் கட்டத்திற்கான உங்கள் தற்போதைய சவால்:',
      'stage_mastered': 'கட்டம் வெற்றிகரமாக முடிக்கப்பட்டது!',
      'all_done':
          'இந்தக் கட்டத்தில் உள்ள அனைத்து சவால்களையும் நீங்கள் முடித்துவிட்டீர்கள்.',
      'return_map': 'வரைபடத்திற்குத் திரும்பு',
      'well_done': 'நன்று செய்தாய்!',
      'i_completed': 'நான் இதை முடித்துவிட்டேன்',
      'level_up_suggestion_title': 'நிலை உயர்வு பரிந்துரை',
      'level_up_suggestion_message':
          'நீங்கள் இந்த நிலை இல் 10 பணிகளை முடித்துவிட்டீர்கள்! நீங்கள் அடுத்த நிலைக்குத் தயாராகிவிட்டீர்கள். அடுத்த நிலைக்குச் செல்ல விரும்புகிறீர்களா?',
      'stay_here': 'இங்கேயே இரு',
      'move_to_next_level': 'அடுத்த நிலைக்கு நகர்த்தவும்',
      // Reflection Screen
      'reflect': 'உங்கள் வளர்ச்சியைப் பற்றி சிந்தியுங்கள்',
      'challenge': 'சவால்',
      'anxiety_q': 'நீங்கள் எவ்வளவு பதற்றமாக உணர்ந்தீர்கள்? (1-10)',
      'what_happened': 'உண்மையில் என்ன நடந்தது?',
      'write_experience_hint': 'உங்கள் அனுபவத்தைப் பற்றி எழுதுங்கள்...',
      'finish': 'சிந்தனையை முடித்துக்கொள்',
      // General / Auth
      'welcome': 'புளூமிற்கு வரவேற்கிறோம்',
      'subtitle': 'உங்கள் தன்னம்பிக்கையை வளர்ப்பதற்கான ஒரு பாதுகாப்பான இடம்.',
      'start': 'எனது பயணத்தைத் தொடங்கு',
      'guest': 'விருந்தினராகத் தொடரவும்',
      'hello': 'வணக்கம்',
      'profile': 'எனது சுயவிவரம்',
      'history': 'எனது வளர்ச்சிப் பயணம்',
      'streak': 'தற்போதைய தொடர் நாட்கள்',
      'best': 'சிறந்த தொடர் நாட்கள்',
      'points': 'தன்னம்பிக்கை புள்ளிகள்',
      //Splash Screen
      'loading': 'உங்கள் தோட்டம் ஏற்றப்படுகிறது...',
    },
    'te': {
      // Progress Screen
      'your_journey': 'నీ ప్రయాణం',
      'keep_growing_sub': 'ప్రతి చిన్న అడుగు ఒక విజయమే. ఎదుగుతూనే ఉండు!',
      'how_it_works': 'ఇది ఎలా పనిచేస్తుంది?',
      'total_points': 'మొత్తం పాయింట్లు',
      'current_streak': 'ప్రస్తుత స్ట్రీక్',
      'tasks_done': 'పూర్తయిన పనులు',
      'rank': 'ర్యాంకు',
      'days': 'రోజులు',
      'contact_us': 'మమ్మల్ని సంప్రదించండి',
      'contact_email_prompt':
          'సహాయం మరియు అభిప్రాయాల కోసం, మాకు ఈమెయిల్ చేయండి:',
      'close': 'మూసివేయి',
      // Level Map Screen
      'tap_to_view_journey': 'నీ ప్రయాణాన్ని చూడటానికి నొక్కండి! 🌸',
      'tap_to_start': 'ఛాలెంజ్ ప్రారంభించడానికి నొక్కండి',
      'choose_level': 'నీ ఎదుగుదల దశను ఎంచుకో:',
      'view_journey': 'ఎదుగుదల ప్రయాణాన్ని చూడు',
      'progress': 'నీ ఎదుగుదల పురోగతి',
      'level_seedling': 'మొక్క (Seedling)',
      'level_sprout': 'మొలక (Sprout)',
      'level_leaf': 'ఆకు (Leaf)',
      'level_stem': 'కాండం (Stem)',
      'level_bloom': 'వికసించడం (Bloom)',
      // Profile Screen
      'account': 'ఖాతా',
      'display_name': 'ప్రదర్శన పేరు',
      'save_name': 'పేరును సేవ్ చేయి',
      'app_theme': 'యాప్ థీమ్',
      'select_color': 'నీ బ్లూమ్ రంగును ఎంచుకో:',
      'light_mode': 'లైట్',
      'dark_mode': 'డార్క్',
      'language': 'భాష',
      'logout': 'లాగౌట్',
      'profile_updated': 'ప్రొఫైల్ అప్‌డేట్ చేయబడింది!',
      'pick_theme_color': 'థీమ్ రంగును ఎంచుకోండి',
      'done': 'పూర్తయింది',
      'reset_all_progress': 'మొత్తం పురోగతిని రీసెట్ చేయి',
      'reset_confirm_title': 'మీరు ఖచ్చితంగా అనుకుంటున్నారా?',
      'reset_confirm_message':
          'ఇది నీ కాన్ఫిడెన్స్ స్కోర్, స్ట్రీక్ మరియు మొత్తం హిస్టరీని శాశ్వతంగా తొలగిస్తుంది. దీన్ని తిరిగి పొందలేము.',
      'cancel': 'రద్దు చేయి',
      'reset_everything': 'అన్నీ రీసెట్ చేయి',
      'reset_success': 'మొత్తం పురోగతి రీసెట్ చేయబడింది.',
      // Shop Screen
      'shop_title': 'బ్లూమ్ షాప్',
      'your_points': 'నీ పాయింట్లు',
      'available_items': 'అందుబాటులో ఉన్న వస్తువులు',
      'streak_freeze': 'స్ట్రీక్ ఫ్రీజ్',
      'protects_streak': 'స్ట్రీక్ రీసెట్ అవ్వకుండా కాపాడుతుంది',
      'your_inventory': 'నీ ఇన్వెంటరీ',
      'owned': 'కొనుగోలు చేసినవి',
      'equipped': 'వాడుకలో ఉంది',
      'equip_freeze': 'ఫ్రీజ్‌ను వాడు',
      'buy': 'కొనుగోలు చేయి',
      'freeze_purchased': 'ఫ్రీజ్ కొనుగోలు చేయబడింది!',
      'not_enough_points': 'సరిపోవు పాయింట్లు లేవు!',
      'freeze_equipped': 'ఫ్రీజ్ యాక్టివేట్ చేయబడింది!',
      // Milestone Screen
      'your_growth_path': 'నీ ఎదుగుదల మార్గం',
      'the_awakening': 'మేల్కొలుపు',
      'seed_badge': 'విత్తనం బ్యాడ్జ్',
      'first_spark': 'తొలి వెలుగు',
      'bronze_leaf': 'కంచు ఆకు',
      'social_courage': 'సామాజిక ధైర్యం',
      'silver_sprout': 'వెండి మొలక',
      'confidence_bloom': 'ఆత్మవిశ్వాస వికాసం',
      'gold_flower': 'బంగారు పువ్వు',
      'mastery': 'నైపుణ్యం',
      'diamond_crown': 'వజ్రాల కిరీటం',
      'milestone_claimed': 'పొందారు',
      'milestone_need_score': '{0}% అవసరం',
      'milestone_claim_reward': 'బహుమతిని పొందు +{0} పాయింట్లు',
      'milestone_reward_toast': 'బహుమతి పొందారు! +{0} పాయింట్లు',
      // FAQ Screen
      'help_faq': 'సహాయం మరియు తరచుగా అడిగే ప్రశ్నలు',
      'common_questions': 'సాధారణ ప్రశ్నలు',
      'keep_blooming': 'వికసిస్తూనే ఉండు! 🌸',
      'faq_q1': 'బ్లూమ్ (Bloom) అంటే ఏమిటి?',
      'faq_a1':
          'బ్లూమ్ అనేది \'గ్రేడెడ్ ఎక్స్‌పోజర్\' అనే ప్రక్రియ ద్వారా ప్రజలలో సామాజిక ఆందోళనను (Social Anxiety) తగ్గించడంలో సహాయపడటానికి రూపొందించబడిన ఒక స్వయం-సహాయ సాధనం. చిన్న చిన్న, సులభమైన సామాజిక పనులను పూర్తి చేయడం ద్వారా, సమాజంలో ఇతరులతో మాట్లాడటం సురక్షితమైనదేనని నువ్వు నీ మెదడుకు అలవాటు చేస్తావు.',
      'faq_q2': 'లెవల్స్ ఎలా పనిచేస్తాయి?',
      'faq_a2':
          'మనం \'మొక్క\' (చాలా సులభమైన పనులు) దశతో ప్రారంభించి, క్రమంగా \'వికసించడం\' (మరింత సవాలుతో కూడిన పనులు) దశకు ఎదుగుతాము. నువ్వు పనులను పూర్తి చేస్తున్నప్పుడు, ఆత్మవిశ్వాస పాయింట్లను పొందుతావు. లెవల్ పెరిగేకొద్దీ, ఎక్కువ పాయింట్లు వస్తాయి!',
      'faq_q3': 'కాన్ఫిడెన్స్ మీటర్ అంటే ఏమిటి?',
      'faq_a3':
          'నీ హోమ్ స్క్రీన్‌పై ఉన్న ప్రోగ్రెస్ బార్ నీ మొత్తం ఆత్మవిశ్వాసాన్ని సూచిస్తుంది. నువ్వు పనులను పూర్తి చేస్తున్న కొద్దీ ఇది పెరుగుతుంది. జాగ్రత్త: నువ్వు కొన్ని రోజుల పాటు ప్రాక్టీస్ చేయడం ఆపేస్తే, నీ కాన్ఫిడెన్స్ స్కోర్ కొద్దిగా తగ్గే అవకాశం ఉంది. ఆత్మవిశ్వాసం అనేది క్రమం తప్పకుండా వ్యాయామం అవసరమయ్యే కండరాల లాంటిదని ఇది నీకు గుర్తు చేస్తుంది!',
      'faq_q4': 'స్ట్రీక్ (Streak) అంటే ఏమిటి?',
      'faq_a4':
          'స్ట్రీక్ అనేది నువ్వు వరుసగా ఎన్ని రోజుల పాటు కనీసం ఒక పనినైనా పూర్తి చేసావు అనేదాని లెక్క. ఆందోళనను అధిగమించడానికి నిలకడగా ఉండటమే ముఖ్యమైన తాళంచెవి, కాబట్టి నీ మనసులోని ఉత్సాహాన్ని నిలిపి ఉంచడానికి ప్రయత్నించు!',
      'faq_q5': 'నా డేటా ఎక్కడ స్టోర్ చేయబడుతుంది?',
      'faq_a5':
          'నీ గోప్యత మా మొదటి ప్రాధాన్యత. నీ పురోగతి, హిస్టరీ మరియు ప్రొఫైల్ డేటా అంతా నీ స్వంత పరికరంలోనే స్థానికంగా (Locally) సేవ్ చేయబడుతుంది. ఏ సమాచారమూ క్లౌడ్ సర్వర్‌కు అప్‌లోడ్ చేయబడదు.',
      'faq_q6': 'యాప్ క్రాష్ అయితే నేను ఏమి చేయాలి?',
      'faq_a6':
          'యాప్ వింతగా ప్రవర్తిస్తుంటే, నీ ఫోన్‌ను రీస్టార్ట్ చేయడానికి ప్రయత్నించు. నువ్వు యాప్‌ను అప్‌డేట్ చేసినట్లయితే, నీ ఆండ్రాయిడ్ సెట్టింగ్స్‌లో యాప్ క్యాష్ (Cache) క్లియర్ చేయాల్సి రావచ్చు. ఏదీ పనిచేయకపోతే, నువ్వు నీ ప్రొఫైల్‌లోని \'మొత్తం పురోగతిని రీసెట్ చేయి\' అనే ఆప్షన్‌ను ఉపయోగించవచ్చు.',
      'faq_q7': 'నేను లెవల్స్‌ను స్కిప్ చేయవచ్చా?',
      'faq_a7':
          'అవును! మేము క్రమంగా ముందుకు సాగాలని సూచించినప్పటికీ, నీ ప్రస్తుత సౌకర్య స్థాయికి తగినట్లుగా మ్యాప్ నుండి ఏ లెవల్‌నైనా ఎంచుకోవడానికి నీకు పూర్తి స్వేచ్ఛ ఉంది.',
      'faq_q8': 'టాస్క్ చాలా కష్టంగా ఉంటే ఏమి చేయాలి?',
      'faq_a8':
          'పనిని పూర్తి చేయడానికి ప్రయత్నించమని మేము సిఫార్సు చేసినప్పటికీ, నువ్వు కేవలం హోమ్ స్క్రీన్‌కి తిరిగి వెళ్లి, ప్రస్తుత టాస్క్‌ను మార్చడానికి మళ్లీ ప్రవేశించవచ్చు.',
      'faq_q9': 'మమ్మల్ని సంప్రదించండి',
      'faq_a9':
          'మా యాప్ గురించి వినియోగదారుల అభిప్రాయాలను మరియు భవిష్యత్ అప్‌డేట్‌ల కోసం సూచనలను తెలుసుకోవడానికి మేము ఇష్టపడతాము. యాప్ ఎంతవరకు ఉపయోగపడుతోంది, ఇందులో ఏముంది లోపించింది మరియు ఎక్కడ మెరుగుదల అవసరం అనేది తెలుసుకోవాలనుకుంటున్నాము. దయచేసి feedback.bloom@gmail.com లో మీ అభిప్రాయాన్ని పంచుకోవడానికి సంకోచించకండి, మేము దానిని ఎంతో అభినందిస్తాము.',
      // Task Screen
      'stage_label': 'దశ: {0}',
      'keep_growing': 'ముందుకు సాగు, {0}',
      'current_challenge': 'ఈ దశకు నీ ప్రస్తుత సవాలు:',
      'stage_mastered': 'దశ విజయవంతంగా పూర్తయింది!',
      'all_done': 'నువ్వు ఈ దశలోని అన్ని సవాళ్లను పూర్తి చేసావు.',
      'return_map': 'మ్యాప్‌కు తిరిగి వెళ్ళు',
      'well_done': 'బాగా చేసావు!',
      'i_completed': 'నేను ఇది పూర్తి చేసాను',
      'level_up_suggestion_title': 'లెవెల్ అప్ సూచన',
      'level_up_suggestion_message':
          'మీరు ఈ స్థాయిలో 10 పనులను పూర్తి చేసారు! మీరు తదుపరి స్థాయికి సిద్ధంగా ఉన్నారు. పైకి వెళ్లాలనుకుంటున్నారా?',
      'stay_here': 'ఇక్కడే ఉండండి',
      'move_to_next_level': 'తదుపరి స్థాయికి తరలించండి',
      // Reflection Screen
      'reflect': 'నీ ఎదుగుదలను సమీక్షించుకో',
      'challenge': 'సవాలు',
      'anxiety_q': 'నువ్వు ఎంత ఆందోళనగా ఫీల్ అయ్యావు? (1-10)',
      'what_happened': 'అసలు ఏమి జరిగింది?',
      'write_experience_hint': 'నీ అనుభవం గురించి రాయి...',
      'finish': 'సమీక్షను పూర్తి చేయి',
      // General / Auth
      'welcome': 'బ్లూమ్‌కి స్వాగతం',
      'subtitle': 'నీ ఆత్మవిశ్వాసాన్ని పెంచుకోవడానికి ఒక సురక్షితమైన స్థలం.',
      'start': 'నా ప్రయాణాన్ని ప్రారంభించు',
      'guest': 'గెస్ట్‌గా కొనసాగించు',
      'hello': 'నమస్తే',
      'profile': 'నా ప్రొఫైల్',
      'history': 'నా ఎదుగుదల ప్రయాణం',
      'streak': 'ప్రస్తుత స్ట్రీక్',
      'best': 'ఉత్తమ స్ట్రీక్',
      'points': 'ఆత్మవిశ్వాస పాయింట్లు',
      //Splash Screen
      'loading': 'నీ తోట లోడ్ అవుతోంది...',
    },
    'kn': {
      // Progress Screen
      'your_journey': 'ನಿನ್ನ ಪಯಣ',
      'keep_growing_sub':
          'ಪ್ರತಿಯೊಂದು ಸಣ್ಣ ಹೆಜ್ಜೆಯೂ ಒಂದು ವಿಜಯ. ಬೆಳೆಯುತ್ತಲೇ ಇರು!',
      'how_it_works': 'ಇದು ಹೇಗೆ ಕೆಲಸ ಮಾಡುತ್ತದೆ?',
      'total_points': 'ಒಟ್ಟು ಪಾಯಿಂಟ್‌ಗಳು',
      'current_streak': 'ಪ್ರಸ್ತುತ ಸ್ಟ್ರೀಕ್',
      'tasks_done': 'ಪೂರ್ಣಗೊಂಡ ಕೆಲಸಗಳು',
      'rank': 'ಶ್ರೇಣಿ (Rank)',
      'days': 'ದಿನಗಳು',
      'contact_us': 'ನಮ್ಮನ್ನು ಸಂಪರ್ಕಿಸಿ',
      'contact_email_prompt':
          'ಬೆಂಬಲ ಮತ್ತು ಪ್ರತಿಕ್ರಿಯೆಗಳಿಗಾಗಿ, ನಮಗೆ ಇಮೇಲ್ ಮಾಡಿ:',
      'close': 'ಮುಚ್ಚು',
      // Level Map Screen
      'tap_to_view_journey': 'ನಿನ್ನ ಪಯಣವನ್ನು ವೀಕ್ಷಿಸಲು ಟ್ಯಾಪ್ ಮಾಡಿ! 🌸',
      'tap_to_start': 'ಸವಾಲನ್ನು ಪ್ರಾರಂಭಿಸಲು ಟ್ಯಾಪ್ ಮಾಡಿ',
      'choose_level': 'ನಿನ್ನ ಬೆಳವಣಿಗೆಯ ಹಂತವನ್ನು ಆರಿಸು:',
      'view_journey': 'ಬೆಳವಣಿಗೆಯ ಪಯಣವನ್ನು ವೀಕ್ಷಿಸು',
      'progress': 'ನಿನ್ನ ಬೆಳವಣಿಗೆಯ ಪ್ರಗತಿ',
      'level_seedling': 'ಸಸಿ (Seedling)',
      'level_sprout': 'ಮೊಳಕೆ (Sprout)',
      'level_leaf': 'ಎಲೆ (Leaf)',
      'level_stem': 'ಕಾಂಡ (Stem)',
      'level_bloom': 'ಅರಳುವುದು (Bloom)',
      // Profile Screen
      'account': 'ಖಾತೆ',
      'display_name': 'ಪ್ರದರ್ಶನದ ಹೆಸರು',
      'save_name': 'ಹೆಸರನ್ನು ಉಳಿಸು',
      'app_theme': 'ಆಪ್ ಥೀಮ್',
      'select_color': 'ನಿನ್ನ ಬ್ಲೂಮ್ ಬಣ್ಣವನ್ನು ಆರಿಸು:',
      'light_mode': 'ಲೈಟ್',
      'dark_mode': 'ಡಾರ್ಕ್',
      'language': 'ಭಾಷೆ',
      'logout': 'ಲಾಗ್ಔಟ್',
      'profile_updated': 'ಪ್ರೊಫೈಲ್ ನವೀಕರಿಸಲಾಗಿದೆ!',
      'pick_theme_color': 'ಥೀಮ್ ಬಣ್ಣವನ್ನು ಆರಿಸಿ',
      'done': 'ಮುಗಿಯಿತು',
      'reset_all_progress': 'ಎಲ್ಲಾ ಪ್ರಗತಿಯನ್ನು ಮರುಹೊಂದಿಸಿ (Reset)',
      'reset_confirm_title': 'ನಿಮಗೆ ಖಚಿತವಾಗಿದೆಯೇ?',
      'reset_confirm_message':
          'ಇದು ನಿನ್ನ ಆತ್ಮವಿಶ್ವಾಸದ ಸ್ಕೋರ್, ಸ್ಟ್ರೀಕ್ ಮತ್ತು ಎಲ್ಲಾ ಇತಿಹಾಸವನ್ನು ಶಾಶ್ವತವಾಗಿ ಅಳಿಸಿಹಾಕುತ್ತದೆ. ಇದನ್ನು ಹಿಂಪಡೆಯಲು ಸಾಧ್ಯವಿಲ್ಲ.',
      'cancel': 'ರದ್ದುಗೊಳಿಸು',
      'reset_everything': 'ಎಲ್ಲವನ್ನೂ ಮರುಹೊಂದಿಸಿ',
      'reset_success': 'ಎಲ್ಲಾ ಪ್ರಗತಿಯನ್ನು ಮರುಹೊಂದಿಸಲಾಗಿದೆ.',
      // Shop Screen
      'shop_title': 'ಬ್ಲೂಮ್ ಶಾಪ್',
      'your_points': 'ನಿನ್ನ ಪಾಯಿಂಟ್‌ಗಳು',
      'available_items': 'ಲಭ್ಯವಿರುವ ವಸ್ತುಗಳು',
      'streak_freeze': 'ಸ್ಟ್ರೀಕ್ ಫ್ರೀಜ್',
      'protects_streak': 'ಸ್ಟ್ರೀಕ್ ಮರುಹೊಂದಿಸದಂತೆ ರಕ್ಷಿಸುತ್ತದೆ',
      'your_inventory': 'ನಿನ್ನ ಇನ್ವೆಂಟರಿ',
      'owned': 'ಖರೀದಿಸಲಾಗಿದೆ',
      'equipped': 'ಬಳಕೆಯಲ್ಲಿದೆ',
      'equip_freeze': 'ಫ್ರೀಜ್ ಅನ್ನು ಬಳಸು',
      'buy': 'ಖರೀದಿಸು',
      'freeze_purchased': 'ಫ್ರೀಜ್ ಖರೀದಿಸಲಾಗಿದೆ!',
      'not_enough_points': 'ಸಾಕಷ್ಟು ಪಾಯಿಂಟ್‌ಗಳಿಲ್ಲ!',
      'freeze_equipped': 'ಫ್ರೀಜ್ ಸಕ್ರಿಯಗೊಳಿಸಲಾಗಿದೆ!',
      // Milestone Screen
      'your_growth_path': 'ನಿನ್ನ ಬೆಳವಣಿಗೆಯ ಹಾದಿ',
      'the_awakening': 'ಜಾಗೃತಿ',
      'seed_badge': 'ಬೀಜದ ಬ್ಯಾಡ್ಜ್',
      'first_spark': 'ಮೊದಲ ಕಿಡಿ',
      'bronze_leaf': 'ಕಂಚಿನ ಎಲೆ',
      'social_courage': 'ಸಾಮಾಜಿಕ ಧೈರ್ಯ',
      'silver_sprout': 'ಬೆಳ್ಳಿಯ ಮೊಳಕೆ',
      'confidence_bloom': 'ಆತ್ಮವಿಶ್ವಾಸದ ವಿಕಾಸ',
      'gold_flower': 'ಚಿನ್ನದ ಹೂವು',
      'mastery': 'ನೈಪುಣ್ಯತೆ',
      'diamond_crown': 'ವಜ್ರದ ಕಿರೀಟ',
      'milestone_claimed': 'ಪಡೆಯಲಾಗಿದೆ',
      'milestone_need_score': '%{0} ಅಗತ್ಯವಿದೆ',
      'milestone_claim_reward': 'ಬಹುಮಾನವನ್ನು ಪಡೆ +{0} ಪಾಯಿಂಟ್‌ಗಳು',
      'milestone_reward_toast': 'ಬಹುಮಾನ ಪಡೆಯಲಾಗಿದೆ! +{0} ಪಾಯಿಂಟ್‌ಗಳು',
      // FAQ Screen
      'help_faq': 'ಸಹಾಯ ಮತ್ತು ತರಚು ಕೇಳಲಾಗುವ ಪ್ರಶ್ನೆಗಳು',
      'common_questions': 'ಸಾಮಾನ್ಯ ಪ್ರಶ್ನೆಗಳು',
      'keep_blooming': 'ಅರಳುತ್ತಲೇ ಇರು! 🌸',
      'faq_q1': 'ಬ್ಲೂಮ್ (Bloom) ಎಂದರೇನು?',
      'faq_a1':
          'ಬ್ಲೂಮ್ ಎನ್ನುವುದು \'ಗ್ರೇಡೆಡ್ ಎಕ್ಸ್‌ಪೋಶರ್\' ಎಂಬ ಪ್ರಕ್ರಿಯೆಯ ಮೂಲಕ ಜನರ ಸಾಮಾಜಿಕ ಆತಂಕವನ್ನು (Social Anxiety) ಕಡಿಮೆ ಮಾಡಲು ಸಹಾಯ ಮಾಡಲು ವಿನ್ಯಾಸಗೊಳಿಸಲಾದ ಸ್ವಯಂ-ಸಹಾಯ ಸಾಧನವಾಗಿದೆ. ಸಣ್ಣ, ಸುಲಭವಾದ ಸಾಮಾಜಿಕ ಕೆಲಸಗಳನ್ನು ಪೂರ್ಣಗೊಳಿಸುವ ಮೂಲಕ, ಸಮಾಜದಲ್ಲಿ ಇತರರೊಂದಿಗೆ ಮಾತನಾಡುವುದು ಸುರಕ್ಷಿತವಾಗಿದೆ ಎಂಬುದನ್ನು ನಿನ್ನ ಮೆದುಳಿಗೆ ನೀನು ರೂಢಿಸಿಕೊಳ್ಳುತ್ತೀಯಾ.',
      'faq_q2': 'ಲೆವೆಲ್‌ಗಳು ಹೇಗೆ ಕೆಲಸ ಮಾಡುತ್ತವೆ?',
      'faq_a2':
          'ನಾವು \'ಸಸಿ\' (ತುಂಬಾ ಸುಲಭವಾದ ಕೆಲಸಗಳು) ಹಂತದಿಂದ ಪ್ರಾರಂಭಿಸಿ, ಕ್ರಮವಾಗಿ \'ಅರಳುವುದು\' (ಹೆಚ್ಚು ಸವಾಲಿನ ಕೆಲಸಗಳು) ಹಂತದವರೆಗೆ ಬೆಳೆಯುತ್ತೇವೆ. ನೀನು ಕೆಲಸಗಳನ್ನು ಪೂರ್ಣಗೊಳಿಸಿದಾಗ, ಆತ್ಮವಿಶ್ವಾಸದ ಪಾಯಿಂಟ್‌ಗಳನ್ನು ಗಳಿಸುತ್ತೀಯಾ. ಲೆವೆಲ್ ಹೆಚ್ಚಾದಂತೆ, ಹೆಚ್ಚು ಪಾಯಿಂಟ್‌ಗಳು ಬರುತ್ತವೆ!',
      'faq_q3': 'ಆತ್ಮವಿಶ್ವಾಸದ ಮೀಟರ್ ಎಂದರೇನು?',
      'faq_a3':
          'ನಿನ್ನ ಹೋಮ್ ಸ್ಕ್ರೀನ್‌ನಲ್ಲಿರುವ ಪ್ರೋಗ್ರೆಸ್ ಬಾರ್ ನಿನ್ನ ಒಟ್ಟಾರೆ ಆತ್ಮವಿಶ್ವಾಸವನ್ನು ಸೂಚಿಸುತ್ತದೆ. ನೀನು ಕೆಲಸಗಳನ್ನು ಪೂರ್ಣಗೊಳಿಸಿದಂತೆ ಇದು ಬೆಳೆಯುತ್ತದೆ. ಜಾಗರೂಕರಾಗಿರಿ: ನೀನು ಕೆಲವು ದಿನಗಳವರೆಗೆ ಅಭ್ಯಾಸ ಮಾಡುವುದನ್ನು ನಿಲ್ಲಿಸಿದರೆ, ನಿನ್ನ ಆತ್ಮವಿಶ್ವಾಸದ ಸ್ಕೋರ್ ಸ್ವಲ್ಪ ಕಡಿಮೆಯಾಗಬಹುದು. ಆತ್ಮವಿಶ್ವಾಸವು ನಿಯಮಿತ ವ್ಯಾಯಾಮದ ಅಗತ್ಯವಿರುವ ಸ್ನಾಯುವಿನಂತಿದೆ ಎಂಬುದನ್ನು ಇದು ನಿನಗೆ ನೆನಪಿಸುತ್ತದೆ!',
      'faq_q4': 'ಸ್ಟ್ರೀಕ್ (Streak) ಎಂದರೇನು?',
      'faq_a4':
          'ಸ್ಟ್ರೀಕ್ ಎಂದರೆ ನೀನು ಸತತವಾಗಿ ಎಷ್ಟು ದಿನಗಳ ಕಾಲ ಕನಿಷ್ಠ ಒಂದು ಕೆಲಸವನ್ನಾದರೂ ಪೂರ್ಣಗೊಳಿಸಿದ್ದೀಯಾ ಎಂಬದರ ಲೆಕ್ಕವಾಗಿದೆ. ಆತಂಕವನ್ನು ಜಯಿಸಲು ಸ್ಥಿರತೆಯೇ ಮುಖ್ಯ ಚಾವಿ, ಆದ್ದರಿಂದ ನಿನ್ನ ಮನಸ್ಸಿನ ಉತ್ಸಾಹವನ್ನು ಉಳಿಸಿಕೊಳ್ಳಲು ಪ್ರಯತ್ನಿಸು!',
      'faq_q5': 'ನನ್ನ ಡೇಟಾವನ್ನು ಎಲ್ಲಿ ಸಂಗ್ರಹಿಸಲಾಗುತ್ತದೆ?',
      'faq_a5':
          'ನಿನ್ನ ಗೌಪ್ಯತೆಯೇ ನಮ್ಮ ಮೊದಲ ಆದ್ಯತೆ. ನಿನ್ನ ಪ್ರಗತಿ, ಇತಿಹಾಸ ಮತ್ತು ಪ್ರೊಫೈಲ್ ಡೇಟಾ ಎಲ್ಲವೂ ನಿನ್ನ ಸ್ವಂತ ಸಾಧನದಲ್ಲೇ ಸ್ಥಳೀಯವಾಗಿ (Locally) ಉಳಿಸಲ್ಪಡುತ್ತದೆ. ಯಾವುದೇ ಮಾಹಿತಿಯು ಕ್ಲೌಡ್ ಸರ್ವರ್‌ಗೆ ಅಪ್‌ಲೋಡ್ ಆಗುವುದಿಲ್ಲ.',
      'faq_q6': 'ಆಪ್ ಕ್ರಾಶ್ ಆದರೆ ನಾನು ಏನು ಮಾಡಬೇಕು?',
      'faq_a6':
          'ಆಪ್ ವಿಚಿತ್ರವಾಗಿ ವರ್ತಿಸುತ್ತಿದ್ದರೆ, ನಿನ್ನ ಫೋನ್ ಅನ್ನು ರಿಸ್ಟಾರ್ಟ್ ಮಾಡಲು ಪ್ರಯತ್ನಿಸು. ನೀನು ಆಪ್ ಅನ್ನು ಅಪ್‌ಡೇಟ್ ಮಾಡಿದ್ದರೆ, ನಿನ್ನ ಆಂಡ್ರಾಯ್ಡ್ ಸೆಟ್ಟಿಂಗ್ಸ್‌ ನಲ್ಲಿ ಆಪ್ ಕ್ಯಾಶ್ (Cache) ಕ್ಲಿಯರ್ ಮಾಡಬೇಕಾಗಬಹುದು. ಯಾವುದೂ ಕೆಲಸ ಮಾಡದಿದ್ದರೆ, ನೀನು ನಿನ್ನ ಪ್ರೊಫೈಲ್‌ನಲ್ಲಿರುವ \'ಎಲ್ಲಾ ಪ್ರಗತಿಯನ್ನು ಮರುಹೊಂದಿಸಿ\' ಎಂಬ ಆಯ್ಕೆಯನ್ನು ಬಳಸಬಹುದು.',
      'faq_q7': 'ನಾನು ಲೆವೆಲ್‌ಗಳನ್ನು ಸ್ಕಿಪ್ ಮಾಡಬಹುದೇ?',
      'faq_a7':
          'ಹೌದು! ನಾವು ಹಂತ ಹಂತವಾಗಿ ಮುಂದುವರಿಯಲು ಸೂಚಿಸಿದರೂ, ನಿನ್ನ ಪ್ರಸ್ತುತ ಸೌಕರ್ಯದ ಮಟ್ಟಕ್ಕೆ ಸೂಕ್ತವೆನಿಸುವ ಯಾವುದೇ ಲೆವೆಲ್ ಅನ್ನು ಮ್ಯಾಪ್‌ನಿಂದ ಆರಿಸಿಕೊಳ್ಳಲು ನಿನಗೆ ಪೂರ್ಣ ಸ್ವಾತಂತ್ರ್ಯವಿದೆ.',
      'faq_q8': 'ಟಾಸ್ಕ್ ತುಂಬಾ ಕಷ್ಟವಾಗಿದ್ದರೆ ಏನು ಮಾಡಬೇಕು?',
      'faq_a8':
          'ಕೆಲಸವನ್ನು ಪೂರ್ಣಗೊಳಿಸಲು ಪ್ರಯತ್ನಿಸಲು ನಾವು ಶಿಫಾರಸು ಮಾಡಿದರೂ, ನೀನು ಕೇವಲ ಹೋಮ್ ಸ್ಕ್ರೀನ್‌ಗೆ ಹಿಂತಿರುಗಿ, ಪ್ರಸ್ತುತ ಟಾಸ್ಕ್ ಅನ್ನು ಬದಲಾಯಿಸಲು ಮತ್ತೊಮ್ಮೆ ಪ್ರವೇಶಿಸಬಹುದು.',
      'faq_q9': 'ನಮ್ಮನ್ನು ಸಂಪರ್ಕಿಸಿ',
      'faq_a9':
          'ನಮ್ಮ ಆಪ್ ಬಗ್ಗೆ ಬಳಕೆದಾರರ ಅಭಿಪ್ರಾಯಗಳನ್ನು ಮತ್ತು ಭವಿಷ್ಯದ ಅಪ್‌ಡೇಟ್‌ಗಳಿಗಾಗಿ ಸಲಹೆಗಳನ್ನು ತಿಳಿಯಲು ನಾವು ಇಷ್ಟಪಡುತ್ತೇವೆ. ಆಪ್ ಎಷ್ಟು ಮಟ್ಟಿಗೆ ಉಪಯುಕ್ತವಾಗಿದೆ, ಇದರಲ್ಲಿ ಏನು ಕೊರತೆಯಿದೆ ಮತ್ತು ಎಲ್ಲಿ ಸುಧಾರಣೆಯ ಅಗತ್ಯವಿದೆ ಎಂಬುದನ್ನು ತಿಳಿಯಲು ಬಯಸುತ್ತೇವೆ. ದಯವಿಟ್ಟು feedback.bloom@gmail.com ನಲ್ಲಿ ನಿಮ್ಮ ಅಭಿಪ್ರಾಯವನ್ನು ಹಂಚಿಕೊಳ್ಳಲು ಹಿಂಜರಿಯಬೇಡಿ, ನಾವು ಅದನ್ನು ತುಂಬಾ ಪ್ರಶಂಸಿಸುತ್ತೇವೆ.',
      // Task Screen
      'stage_label': 'ಹಂತ: {0}',
      'keep_growing': 'ಮುಂದೆ ಸಾಗಿ, {0}',
      'current_challenge': 'ಈ ಹಂತಕ್ಕೆ ನಿನ್ನ ಪ್ರಸ್ತುತ ಸವಾಲು:',
      'stage_mastered': 'ಹಂತವು ಯಶಸ್ವಿಯಾಗಿ ಪೂರ್ಣಗೊಂಡಿದೆ!',
      'all_done': 'ನೀನು ಈ ಹಂತದ ಎಲ್ಲಾ ಸವಾಲುಗಳನ್ನು ಪೂರ್ಣಗೊಳಿಸಿದ್ದೀಯಾ.',
      'return_map': 'ಮ್ಯಾಪ್‌ಗೆ ಹಿಂತಿರುಗು',
      'well_done': 'ಒಳ್ಳೆಯ ಕೆಲಸ!',
      'i_completed': 'ನಾನು ಇದನ್ನು ಪೂರ್ಣಗೊಳಿಸಿದ್ದೇನೆ',
      'level_up_suggestion_title': 'ಲೆವೆಲ್ ಅಪ್ ಸಲಹೆ',
      'level_up_suggestion_message':
          'ನೀವು ಈ ಮಟ್ಟದ ನಲ್ಲಿ 10 ಕಾರ್ಯಗಳನ್ನು ಪೂರ್ಣಗೊಳಿಸಿದ್ದೀರಿ! ನೀವು ಮುಂದಿನ ಹಂತಕ್ಕೆ ಸಿದ್ಧರಿದ್ದೀರಿ. ಮೇಲಕ್ಕೆ ಹೋಗಲು ಬಯಸುವಿರಾ?',
      'stay_here': 'ಇಲ್ಲೇ ಇರಿ',
      'move_to_next_level': 'ಮುಂದಿನ ಹಂತಕ್ಕೆ ಸರಿಸಿ',
      // Reflection Screen
      'reflect': 'ನಿನ್ನ ಬೆಳವಣಿಗೆಯನ್ನು ಪರಾಮರ್ಶಿಸು',
      'challenge': 'ಸವಾಲು',
      'anxiety_q': 'ನೀನು ಎಷ್ಟು ಆತಂಕವನ್ನು ಅನುಭವಿಸಿದೆ? (1-10)',
      'what_happened': 'ನಿಜವಾಗಿ ಏನಾಯಿತು?',
      'write_experience_hint': 'ನಿನ್ನ ಅನುಭವದ ಬಗ್ಗೆ ಬರೆ...',
      'finish': 'ಪರಾಮರ್ಶೆಯನ್ನು ಪೂರ್ಣಗೊಳಿಸು',
      // General / Auth
      'welcome': 'ಬ್ಲೂಮ್‌ಗೆ ಸುಸ್ವಾಗತ',
      'subtitle': 'ನಿನ್ನ ಆತ್ಮವಿಶ್ವಾಸವನ್ನು ಬೆಳೆಸಿಕೊಳ್ಳಲು ಒಂದು ಸುರಕ್ಷಿತ ಸ್ಥಳ.',
      'start': 'ನನ್ನ ಪಯಣವನ್ನು ಪ್ರಾರಂಭಿಸು',
      'guest': 'ಗೆಸ್ಟ್ ಆಗಿ ಮುಂದುವರಿಯಿರಿ',
      'hello': 'ನಮಸ್ಕಾರ',
      'profile': 'ನನ್ನ ಪ್ರೊಫೈಲ್',
      'history': 'ನನ್ನ ಬೆಳವಣಿಗೆಯ ಪಯಣ',
      'streak': 'ಪ್ರಸ್ತುತ ಸ್ಟ್ರೀಕ್',
      'best': 'ಉತ್ತಮ ಸ್ಟ್ರೀಕ್',
      'points': 'ಆತ್ಮವಿಶ್ವಾಸದ ಪಾಯಿಂಟ್‌ಗಳು',
      //Splash Screen
      'loading': 'ನಿನ್ನ ತೋಟ ಲೋಡ್ ಆಗುತ್ತಿದೆ...',
    },
    'ml': {
      // Progress Screen
      'your_journey': 'നിങ്ങളുടെ യാത്ര',
      'keep_growing_sub':
          'ഓരോ ചെറിയ ചുവടും ഒരു വിജയമാണ്. വളർന്നു കൊണ്ടേയിരിക്കൂ!',
      'how_it_works': 'ഇത് എങ്ങനെ പ്രവർത്തിക്കുന്നു?',
      'total_points': 'ആകെ പോയിന്റുകൾ',
      'current_streak': 'നിലവിലെ തുടർച്ച (Streak)',
      'tasks_done': 'പൂർത്തിയാക്കിയവ',
      'rank': 'റാങ്ക്',
      'days': 'ദിവസങ്ങൾ',
      'contact_us': 'ഞങ്ങളെ ബന്ധപ്പെടുക',
      'contact_email_prompt':
          'പിന്തുണയ്ക്കും ഫീഡ്‌ബാക്കിനും ഞങ്ങൾക്ക് ഇമെയിൽ ചെയ്യുക:',
      'close': 'അടയ്ക്കുക',
      // Level Map Screen
      'tap_to_view_journey': 'നിങ്ങളുടെ യാത്ര കാണാൻ ടാപ്പ് ചെയ്യുക! 🌸',
      'tap_to_start': 'ചലഞ്ച് ആരംഭിക്കാൻ ടാപ്പ് ചെയ്യുക',
      'choose_level': 'നിങ്ങളുടെ വളർച്ചാ ഘട്ടം തിരഞ്ഞെടുക്കുക:',
      'view_journey': 'വളർച്ചാ യാത്ര കാണുക',
      'progress': 'നിങ്ങളുടെ വളർച്ചാ പുരോഗതി',
      'level_seedling': 'തൈച്ചെടി (Seedling)',
      'level_sprout': 'മുള (Sprout)',
      'level_leaf': 'ഇല (Leaf)',
      'level_stem': 'തണ്ട് (Stem)',
      'level_bloom': 'പൂർണ്ണ വികാസം (Bloom)',
      // Profile Screen
      'account': 'അക്കൗണ്ട്',
      'display_name': 'പ്രദർശിപ്പിക്കുന്ന പേര്',
      'save_name': 'പേര് സംരക്ഷിക്കുക',
      'app_theme': 'ആപ്പ് തീം',
      'select_color': 'നിങ്ങളുടെ ബ്ലൂം നിറം തിരഞ്ഞെടുക്കുക:',
      'light_mode': 'ലൈറ്റ്',
      'dark_mode': 'ഡാർക്ക്',
      'language': 'ഭാഷ',
      'logout': 'ലോഗ്ഔട്ട്',
      'profile_updated': 'പ്രൊഫൈൽ പുതുക്കി!',
      'pick_theme_color': 'തീം നിറം തിരഞ്ഞെടുക്കുക',
      'done': 'പൂർത്തിയായി',
      'reset_all_progress': 'എല്ലാ പുരോഗതിയും റീസെറ്റ് ചെയ്യുക',
      'reset_confirm_title': 'നിങ്ങൾക്ക് ഉറപ്പാണോ?',
      'reset_confirm_message':
          'ഇത് നിങ്ങളുടെ ആത്മവിശ്വാസ സ്‌കോറും, സ്ട്രൈക്കും, എല്ലാ ചരിത്രവും ശാശ്വതമായി ഇല്ലാതാക്കും. ഇത് പഴയപടിയാക്കാൻ കഴിയില്ല.',
      'cancel': 'റദ്ദാക്കുക',
      'reset_everything': 'എല്ലാം റീസെറ്റ് ചെയ്യുക',
      'reset_success': 'എല്ലാ പുരോഗതിയും റീസെറ്റ് ചെയ്തു.',
      // Shop Screen
      'shop_title': 'ബ്ലൂം ഷോപ്പ്',
      'your_points': 'നിങ്ങളുടെ പോയിന്റുകൾ',
      'available_items': 'ലഭ്യമായ ഇനങ്ങൾ',
      'streak_freeze': 'സ്ട്രൈക്ക് ഫ്രീസ്',
      'protects_streak': 'സ്ട്രൈക്ക് റീസെറ്റ് ആകുന്നത് തടയുന്നു',
      'your_inventory': 'നിങ്ങളുടെ ഇൻവെന്ററി',
      'owned': 'വാങ്ങിയവ',
      'equipped': 'ഉപയോഗത്തിലുള്ളത്',
      'equip_freeze': 'ഫ്രീസ് ഉപയോഗിക്കുക',
      'buy': 'വാങ്ങുക',
      'freeze_purchased': 'ഫ്രീസ് വാങ്ങിയിരിക്കുന്നു!',
      'not_enough_points': 'ആവശ്യത്തിന് പോയിന്റുകൾ ഇല്ല!',
      'freeze_equipped': 'ഫ്രീസ് സജീവമാക്കിയിരിക്കുന്നു!',
      // Milestone Screen
      'your_growth_path': 'നിങ്ങളുടെ വളർച്ചാ പാത',
      'the_awakening': 'ഉണർവ്',
      'seed_badge': 'വിത്ത് ബാഡ്ജ്',
      'first_spark': 'ആദ്യ സ്ഫുരണം',
      'bronze_leaf': 'വെങ്കല ഇല',
      'social_courage': 'സാമൂഹിക ധൈര്യം',
      'silver_sprout': 'വെള്ളി മുള',
      'confidence_bloom': 'ആത്മവിശ്വാസത്തിന്റെ വികാസം',
      'gold_flower': 'സ്വർണ്ണ പൂവ്',
      'mastery': 'മാസ്റ്ററി',
      'diamond_crown': 'വൈര കിരീടം',
      'milestone_claimed': 'നേടിയെടുത്തു',
      'milestone_need_score': '{0}% ആവശ്യമുണ്ട്',
      'milestone_claim_reward': 'സമ്മാനം നേടൂ +{0} പോയിന്റുകൾ',
      'milestone_reward_toast': 'സമ്മാനം നേടി! +{0} പോയിന്റുകൾ',
      // FAQ Screen
      'help_faq': 'സഹായവും പൊതുവായ ചോദ്യങ്ങളും',
      'common_questions': 'പൊതുവായ ചോദ്യങ്ങൾ',
      'keep_blooming': 'തുടർന്നും വികസിക്കൂ! 🌸',
      'faq_q1': 'എന്താണ് ബ്ലൂം (Bloom)?',
      'faq_a1':
          '\'ഗ്രേഡഡ് എക്സ്പോഷർ\' എന്ന പ്രക്രിയയിലൂടെ ആളുകളിൽ സാമൂഹിക ഉത്കണ്ഠ (Social Anxiety) കുറയ്ക്കാൻ സഹായിക്കുന്നതിനായി രൂപകൽപ്പന ചെയ്ത ഒരു സ്വയം സഹായ ഉപകരണമാണ് ബ്ലൂം. ചെറിയ, എളുപ്പമുള്ള സാമൂഹിക കാര്യങ്ങൾ പൂർത്തിയാക്കുന്നതിലൂടെ, സാമൂഹിക ഇടപെടലുകൾ സുരക്ഷിതവും കൈകാര്യം ചെയ്യാവുന്നതുമാണെന്ന് നിങ്ങൾ നിങ്ങളുടെ തലച്ചോറിനെ പരിശീലിപ്പിക്കുന്നു.',
      'faq_q2': 'ലെവലുകൾ എങ്ങനെയാണ് പ്രവർത്തിക്കുന്നത്?',
      'faq_a2':
          'നാം \'തൈച്ചെടി\' (വളരെ എളുപ്പമുള്ള ജോലികൾ) എന്നതിൽ തുടങ്ങി, \'പൂർണ്ണ വികാസം\' (കൂടുതൽ വെല്ലുവിളി നിറഞ്ഞ ജോലികൾ) വരെ മുന്നേറുന്നു. നിങ്ങൾ ജോലികൾ പൂർത്തിയാക്കുമ്പോൾ, ആത്മവിശ്വാസ പോയിന്റുകൾ നേടും. ലെവൽ കൂടുന്തോറും കൂടുതൽ പോയിന്റുകൾ നേടാം!',
      'faq_q3': 'കോൺഫിഡൻസ് മീറ്റർ എന്നാൽ എന്താണ്?',
      'faq_a3':
          'നിങ്ങളുടെ ഹോം സ്ക്രീനിലെ പ്രോഗ്രസ് ബാർ നിങ്ങളുടെ മൊത്തത്തിലുള്ള ആത്മവിശ്വാസത്തെ പ്രതിനിധീകരിക്കുന്നു. നിങ്ങൾ ടാസ്ക്കുകൾ പൂർത്തിയാക്കുമ്പോൾ അത് വളരുന്നു. ശ്രദ്ധിക്കുക: നിങ്ങൾ കുറച്ച് ദിവസത്തേക്ക് പരിശീലനം നിർത്തുകയാണെങ്കിൽ, നിങ്ങളുടെ ആത്മവിശ്വാസ സ്കോർ അല്പം കുറഞ്ഞേക്കാം, ആത്മവിശ്വാസം എന്നത് പതിവ് വ്യായാമം ആവശ്യമായ ഒരു പേശിയെപ്പോലെയാണെന്ന് ഇത് നിങ്ങളെ ഓർമ്മിപ്പിക്കുന്നു!',
      'faq_q4': 'തുടർച്ച അല്ലെങ്കിൽ സ്ട്രൈക്ക് (Streak) എന്നാൽ എന്താണ്?',
      'faq_a4':
          'സ്ട്രൈക്ക് എന്നത് നിങ്ങൾ തുടർച്ചയായി എത്ര ദിവസങ്ങൾ കുറഞ്ഞത് ഒരു ടാസ്ക്കെങ്കിലും പൂർത്തിയാക്കി എന്നതിന്റെ കണക്കാണ്. ഉത്കണ്ഠയെ മറികടക്കാൻ സ്ഥിരതയാണ് പ്രധാനം, അതിനാൽ നിങ്ങളുടെ ഉള്ളിലെ തീജ്വാല നിലനിർത്താൻ ശ്രമിക്കുക!',
      'faq_q5': 'എന്റെ ഡാറ്റ എവിടെയാണ് സംഭരിച്ചിരിക്കുന്നത്?',
      'faq_a5':
          'നിങ്ങളുടെ സ്വകാര്യതയ്ക്കാണ് ഞങ്ങളുടെ പ്രഥമ പരിഗണന. നിങ്ങളുടെ എല്ലാ പുരോഗതിയും, ചരിത്രവും, പ്രൊഫൈൽ ഡാറ്റയും നിങ്ങളുടെ സ്വന്തം ഉപകരണത്തിൽ പ്രാദേശികമായി (Locally) മാത്രമേ സംഭരിക്കപ്പെടുകയുള്ളൂ. ക്ലൗഡ് സെർവറിലേക്ക് ഒന്നും അപ്‌ലോഡ് ചെയ്യുന്നില്ല.',
      'faq_q6': 'ആപ്പ് ക്രാഷ് ആയാൽ ഞാൻ എന്ത് ചെയ്യണം?',
      'faq_a6':
          'ആപ്പ് അസാധാരണമായി പെരുമാറുകയാണെങ്കിൽ നിങ്ങളുടെ ഫോൺ റീസ്റ്റാർട്ട് ചെയ്യാൻ ശ്രമിക്കുക. നിങ്ങൾ ആപ്പ് അപ്‌ഡേറ്റ് ചെയ്‌തിട്ടുണ്ടെങ്കിൽ, നിങ്ങളുടെ ആൻഡ്രോയിഡ് ക്രമീകരണങ്ങളിൽ ആപ്പ് കാഷെ (Cache) മായ്‌ക്കേണ്ടി വന്നേക്കാം. ഒന്നും പ്രവർത്തിക്കുന്നില്ലെങ്കിൽ, നിങ്ങളുടെ പ്രൊഫൈലിലെ \'എല്ലാ പുരോഗതിയും റീസെറ്റ് ചെയ്യുക\' എന്ന ഓപ്ഷൻ ഉപയോഗിക്കാം.',
      'faq_q7': 'എനിക്ക് ലെവലുകൾ ഒഴിവാക്കാമോ (Skip)?',
      'faq_a7':
          'അതെ, സാധിക്കും! ഘട്ടം ഘട്ടമായുള്ള പാതയാണ് ഞങ്ങൾ ശുപാർശ ചെയ്യുന്നതെങ്കിലും, നിങ്ങളുടെ നിലവിലെ കംഫർട്ട് ലെവലിന് അനുയോജ്യമെന്ന് തോന്നുന്ന ഏത് ലെവലും മാപ്പിൽ നിന്ന് തിരഞ്ഞെടുക്കാൻ നിങ്ങൾക്ക് പൂർണ്ണ സ്വാതന്ത്ര്യമുണ്ട്.',
      'faq_q8': 'ടാസ്ക് വളരെ ബുദ്ധിമുട്ടാണെങ്കിൽ എന്ത് ചെയ്യും?',
      'faq_a8':
          'ടാസ്ക് പൂർത്തിയാക്കാൻ ശ്രമിക്കാൻ ഞങ്ങൾ ശുപാർശ ചെയ്യുന്നുണ്ടെങ്കിലും, നിങ്ങൾക്ക് ഹോം സ്ക്രീനിലേക്ക് മടങ്ങിപ്പോകാനും നിലവിലെ ടാസ്ക് മാറ്റാൻ വീണ്ടും പ്രവേശിക്കാനും കഴിയും.',
      'faq_q9': 'ഞങ്ങളെ ബന്ധപ്പെടുക',
      'faq_a9':
          'ഞങ്ങളുടെ ഉപയോക്താക്കളിൽ നിന്ന് ആപ്പിനെക്കുറിച്ചുള്ള അഭിപ്രായങ്ങളും ഭാവിയിലെ അപ്‌ഡേറ്റുകൾക്കായുള്ള നിർദ്ദേശങ്ങളും കേൾക്കാൻ ഞങ്ങൾ ആഗ്രഹിക്കുന്നു. ഉപയോക്താക്കൾക്കായി ആപ്പ് എത്രത്തോളം നന്നായി പ്രവർത്തിക്കുന്നുവെന്നും അതിൽ എന്തൊക്കെ കുറവുകളുണ്ടെന്നും എവിടെയൊക്കെ പുരോഗതി ആവശ്യമാണെന്നും അറിയാൻ ഞങ്ങൾ ആഗ്രഹിക്കുന്നു. ദയവായി feedback.bloom@gmail.com ൽ നിങ്ങളുടെ അഭിപ്രായം പങ്കിടാൻ മടിക്കരുത്, ഞങ്ങൾ അതിനെ വളരെയധികം അഭിനന്ദിക്കും.',
      // Task Screen
      'stage_label': 'ഘട്ടം: {0}',
      'keep_growing': 'മുന്നോട്ട് പോകുക, {0}',
      'current_challenge': 'ഈ ഘട്ടത്തിനായുള്ള നിങ്ങളുടെ നിലവിലെ ചലഞ്ച്:',
      'stage_mastered': 'ഘട്ടം വിജയകരമായി പൂർത്തിയാക്കി!',
      'all_done':
          'നിങ്ങൾ ഈ ഘട്ടത്തിലെ എല്ലാ ചലഞ്ചുകളും പൂർത്തിയാക്കിയിരിക്കുന്നു.',
      'return_map': 'മാപ്പിലേക്ക് തിരികെ പോകുക',
      'well_done': 'നന്നായി ചെയ്തു!',
      'i_completed': 'ഞാൻ ഇത് പൂർത്തിയാക്കി',
      'level_up_suggestion_title': 'ലെവൽ അപ്പ് നിർദ്ദേശം',
      'level_up_suggestion_message':
          'ഈ ലെവൽ-ൽ നിങ്ങൾ 10 ടാസ്‌ക്കുകൾ പൂർത്തിയാക്കി! അടുത്ത ലെവലിനായി നിങ്ങൾ തയ്യാറാണ്. മുകളിലേക്ക് പോകാൻ ആഗ്രഹിക്കുന്നുണ്ടോ?',
      'stay_here': 'ഇവിടെ താമസിക്കുക',
      'move_to_next_level': 'അടുത്ത ലെവലിലേക്ക് നീങ്ങുക',
      // Reflection Screen
      'reflect': 'നിങ്ങളുടെ വളർച്ചയെക്കുറിച്ച് ചിന്തിക്കുക',
      'challenge': 'വെല്ലുവിളി',
      'anxiety_q': 'നിങ്ങൾക്ക് എത്രത്തോളം ഉത്കണ്ഠ തോന്നി? (1-10)',
      'what_happened': 'യഥാർത്ഥത്തിൽ എന്താണ് സംഭവിച്ചത്?',
      'write_experience_hint': 'നിങ്ങളുടെ അനുഭവത്തെക്കുറിച്ച് എഴുതുക...',
      'finish': 'ചിന്ത പൂർത്തിയാക്കുക',
      // General / Auth
      'welcome': 'ബ്ലൂമിലേക്ക് സ്വാഗതം',
      'subtitle': 'നിങ്ങളുടെ ആത്മവിശ്വാസം വളർത്തുന്നതിനുള്ള ഒരു സുരക്ഷിത ഇടം.',
      'start': 'എന്റെ യാത്ര ആരംഭിക്കുക',
      'guest': 'അതിഥിയായി തുടരുക',
      'hello': 'നമസ്കാരം',
      'profile': 'എന്റെ പ്രൊഫൈൽ',
      'history': 'എന്റെ വളർച്ചാ യാത്ര',
      'streak': 'നിലവിലെ തുടർച്ച',
      'best': 'മികച്ച തുടർച്ച',
      'points': 'ആത്മവിശ്വാസ പോയിന്റുകൾ',
      //Splash Screen
      'loading': 'നിങ്ങളുടെ തോട്ടം ലോഡ് ചെയ്യുന്നു...',
    },
    'mr': {
      // Progress Screen
      'your_journey': 'तुझा प्रवास',
      'keep_growing_sub': 'प्रत्येक लहान पाऊल हा एक विजय आहे. वाढत राहा!',
      'how_it_works': 'हे कसे काम करते?',
      'total_points': 'एकूण गुण',
      'current_streak': 'सध्याची सातत्यता (Streak)',
      'tasks_done': 'पूर्ण झालेली आव्हाने',
      'rank': 'रँक',
      'days': 'दिवस',
      'contact_us': 'आमच्याशी संपर्क साधा',
      'contact_email_prompt': 'मदत आणि अभिप्रायासाठी आम्हाला येथे ईमेल करा:',
      'close': 'बंद करा',
      // Level Map Screen
      'tap_to_view_journey': 'तुझा प्रवास पाहण्यासाठी टॅप कर! 🌸',
      'tap_to_start': 'आव्हान सुरू करण्यासाठी टॅप कर',
      'choose_level': 'तुझा प्रगतीचा टप्पा निवडा:',
      'view_journey': 'प्रगतीचा प्रवास पहा',
      'progress': 'तुझी प्रगती',
      'level_seedling': 'रोपटे (Seedling)',
      'level_sprout': 'अंकुर (Sprout)',
      'level_leaf': 'पान (Leaf)',
      'level_stem': 'देठ (Stem)',
      'level_bloom': 'बहरणे (Bloom)',
      // Profile Screen
      'account': 'खाते',
      'display_name': 'दिसणारे नाव',
      'save_name': 'नाव सेव्ह करा',
      'app_theme': 'अॅप थीम',
      'select_color': 'तुझा ब्लूम रंग निवडा:',
      'light_mode': 'लाइट',
      'dark_mode': 'डार्क',
      'language': 'भाषा',
      'logout': 'लॉगआउट',
      'profile_updated': 'प्रोफाइल अपडेट झाली!',
      'pick_theme_color': 'थीमचा रंग निवडा',
      'done': 'पूर्ण',
      'reset_all_progress': 'सर्व प्रगती रीसेट करा',
      'reset_confirm_title': 'तुला खात्री आहे का?',
      'reset_confirm_message':
          'यामुळे तुमचे आत्मविश्वासाचे गुण, सातत्यता (Streak) आणि संपूर्ण इतिहास कायमचा मिटवला जाईल. हे पुन्हा मिळवता येणार नाही.',
      'cancel': 'रद्द करा',
      'reset_everything': 'सर्व काही रीसेट करा',
      'reset_success': 'सर्व प्रगती रीसेट केली गेली आहे.',
      // Shop Screen
      'shop_title': 'ब्लूम शॉप',
      'your_points': 'तुझे गुण',
      'available_items': 'उपलब्ध वस्तू',
      'streak_freeze': 'स्ट्रिक फ्रीज',
      'protects_streak': 'सातत्यता (Streak) रीसेट होण्यापासून वाचवते',
      'your_inventory': 'तुझ्या वस्तू (Inventory)',
      'owned': 'खरेदी केले',
      'equipped': 'वापरात आहे',
      'equip_freeze': 'फ्रीज वापरा',
      'buy': 'खरेदी करा',
      'freeze_purchased': 'फ्रीज खरेदी केले!',
      'not_enough_points': 'अपुरे गुण आहेत!',
      'freeze_equipped': 'फ्रीज सक्रिय केले गेले आहे!',
      // Milestone Screen
      'your_growth_path': 'तुझा प्रगतीचा मार्ग',
      'the_awakening': 'जागृती',
      'seed_badge': 'बीज बॅज',
      'first_spark': 'पहिली ठिणगी',
      'bronze_leaf': 'कांस्य पान',
      'social_courage': 'सामाजिक धैर्य',
      'silver_sprout': 'रौप्य अंकुर',
      'confidence_bloom': 'आत्मविश्वासाचा बहर',
      'gold_flower': 'सुवर्ण फूल',
      'mastery': 'मास्टरी',
      'diamond_crown': 'हिऱ्याचा मुकुट',
      'milestone_claimed': 'मिळाले',
      'milestone_need_score': '{0}% आवश्यक',
      'milestone_claim_reward': 'बक्षीस घ्या +{0} गुण',
      'milestone_reward_toast': 'बक्षीस मिळाले! +{0} गुण',
      // FAQ Screen
      'help_faq': 'मदत आणि FAQ',
      'common_questions': 'वारंवार विचारले जाणारे प्रश्न',
      'keep_blooming': 'बहरत राहा! 🌸',
      'faq_q1': 'ब्लूम (Bloom) म्हणजे काय?',
      'faq_a1':
          'ब्लूम हे एक सेल्फ-हेल्प टूल आहे जे \'ग्रॅडेड एक्सपोजर\' नावाच्या प्रक्रियेद्वारे लोकांची सामाजिक भीती (Social Anxiety) कमी करण्यासाठी डिझाइन केलेले आहे. लहान, हाताळण्याजोगी सामाजिक आव्हाने पूर्ण करून, तुम्ही तुमच्या मेंदूला हे पटवून देता की सामाजिक संवाद सुरक्षित आणि सामान्य आहेत.',
      'faq_q2': 'पातळ्या (Levels) कशा काम करतात?',
      'faq_a2':
          'आम्ही \'रोपटे\' (खूप सोपी आव्हाने) पासून सुरुवात करतो आणि हळूहळू \'बहरणे\' (अधिक आव्हानात्मक कामे) कडे जातो. तुम्ही आव्हाने पूर्ण करताच, तुम्हाला आत्मविश्वासाचे गुण मिळतील. पातळी जितकी जास्त असेल, तितके जास्त गुण मिळतील!',
      'faq_q3': 'कॉन्फिडेंस मीटर म्हणजे काय?',
      'faq_a3':
          'तुमच्या होम स्क्रीनवरील प्रोग्रेस बार तुमचा एकूण आत्मविश्वास दर्शवतो. तुम्ही कामे पूर्ण करताच तो वाढतो. सावध राहा: तुम्ही काही दिवस सराव करणे थांबवल्यास, तुमचा आत्मविश्वास स्कोअर थोडा कमी होऊ शकतो, जो तुम्हाला आठवण करून देतो की आत्मविश्वास हा एका स्नायूसारखा आहे ज्याला नियमित व्यायामाची गरज असते!',
      'faq_q4': 'सातत्यता किंवा स्ट्रिक (Streak) म्हणजे काय?',
      'faq_a4':
          'स्ट्रिक म्हणजे तुम्ही सलग किती दिवस किमान एक काम पूर्ण केले आहे याची मोजणी होय. भीतीवर मात करण्यासाठी सातत्य हीच मुख्य गुरुकिल्ली आहे, म्हणून तुमच्यातील उत्साह टिकवून ठेवण्याचा प्रयत्न करा!',
      'faq_q5': 'माझा डेटा कुठे स्टोअर केला जातो?',
      'faq_a5':
          'तुमची गोपनीयता ही आमची पहिली प्राथमिकता आहे. तुमची सर्व प्रगती, इतिहास आणि प्रोफाइल डेटा तुमच्या स्वतःच्या डिव्हाइसवर स्थानिक पातळीवर (Locally) सेव्ह केला जातो. क्लाउड सर्व्हरवर कोणतीही माहिती अपलोड केली जात नाही.',
      'faq_q6': 'अॅप क्रॅश झाल्यास मी काय करू?',
      'faq_a6':
          'अॅप विचित्र वागत असल्यास, तुमचा फोन रीस्टार्ट करण्याचा प्रयत्न करा. तुम्ही अॅप अपडेट केले असल्यास, तुम्हाला तुमच्या अँड्रॉइड सेटिंग्समध्ये अॅप कॅश (Cache) क्लिअर करावे लागेल. काहीही काम करत नसल्यास, तुम्ही तुमच्या प्रोफाइलमधील \'सर्व प्रगती रीसेट करा\' हा पर्याय वापरू शकता.',
      'faq_q7': 'मी लेव्हल्स सोडू (Skip) शकतो का?',
      'faq_a7':
          'होय! आम्ही टप्प्याटप्प्याने पुढे जाण्याचा सल्ला देत असलो तरी, तुमच्या सध्याच्या कम्फर्ट लेव्हलनुसार मॅपमधून कोणतीही लेव्हल निवडण्याचे तुम्हाला पूर्ण स्वातंत्र्य आहे.',
      'faq_q8': 'काम खूप कठीण वाटल्यास काय करावे?',
      'faq_a8':
          'आम्ही तुम्हाला काम पूर्ण करण्याचा प्रयत्न करण्याचा सल्ला देत असलो तरी, तुम्ही फक्त होम स्क्रीनवर परत जाऊ शकता आणि सध्याचे काम बदलण्यासाठी पुन्हा प्रवेश करू शकता.',
      'faq_q9': 'आमच्याशी संपर्क साधा',
      'faq_a9':
          'आम्हाला आमच्या वापरकर्त्यांकडून अॅपबद्दलचे अभिप्राय आणि भविष्यातील अपडेट्ससाठी त्यांच्या सूचना जाणून घ्यायला आवडेल. अॅप वापरकर्त्यांसाठी किती चांगले काम करत आहे, त्यात कशाची कमतरता आहे आणि कुठे सुधारणा आवश्यक आहे हे आम्हाला जाणून घ्यायचे आहे. कृपया feedback.bloom@gmail.com वर तुमचे मत मांडण्यास संकोच करू नका, आम्ही त्याचे मनापासून कौतुक करू.',
      // Task Screen
      'stage_label': 'टप्पा: {0}',
      'keep_growing': 'पुढे चालत राहा, {0}',
      'current_challenge': 'या टप्प्यासाठी तुमचे सध्याचे आव्हान:',
      'stage_mastered': 'टप्पा यशस्वीरीत्या पूर्ण झाला!',
      'all_done': 'तुम्ही या टप्प्यातील सर्व आव्हाने पूर्ण केली आहेत.',
      'return_map': 'मॅपवर परत जा',
      'well_done': 'खूप छान!',
      'i_completed': 'मी हे पूर्ण केले',
      'level_up_suggestion_title': 'पातळी वाढवण्याची सूचना',
      'level_up_suggestion_message':
          'तुम्ही ही पातळी मध्ये १० कार्ये पूर्ण केली आहेत! तुम्ही पुढील स्तरासाठी तयार आहात. वर जायचे आहे का?',
      'stay_here': 'इथेच रहा',
      'move_to_next_level': 'पुढील स्तरावर जा',
      // Reflection Screen
      'reflect': 'तुमच्या प्रगतीचे पुनरावलोकन करा',
      'challenge': 'आव्हान',
      'anxiety_q': 'तुला किती भीती वाटली? (1-10)',
      'what_happened': 'खरोखर काय घडले?',
      'write_experience_hint': 'तुझ्या अनुभवाविषयी लिही...',
      'finish': 'पुनरावलोकन पूर्ण करा',
      // General / Auth
      'welcome': 'ब्लूममध्ये स्वागत आहे',
      'subtitle': 'तुमचा आत्मविश्वास वाढवण्यासाठी एक सुरक्षित जागा.',
      'start': 'माझा प्रवास सुरू करा',
      'guest': 'गेस्ट म्हणून पुढे जा',
      'hello': 'नमस्कार',
      'profile': 'माझी प्रोफाइल',
      'history': 'माझा प्रगतीचा प्रवास',
      'streak': 'सध्याची सातत्यता',
      'best': 'सर्वोत्तम सातत्यता',
      'points': 'आत्मविश्वास गुण',
      //Splash Screen
      'loading': 'तुमची बाग लोड होत आहे...',
    },
    'pt': {
      // Progress Screen
      'your_journey': 'Tua Jornada',
      'keep_growing_sub':
          'Cada pequeno passo é uma vitória. Continua a crescer!',
      'how_it_works': 'Como funciona?',
      'total_points': 'Total de Pontos',
      'current_streak': 'Série Atual',
      'tasks_done': 'Tarefas Concluídas',
      'rank': 'Classificação',
      'days': 'Dias',
      'contact_us': 'Contacta-nos',
      'contact_email_prompt':
          'Para suporte e feedback, envia-nos um e-mail para:',
      'close': 'Fechar',
      // Level Map Screen
      'tap_to_view_journey': 'Toca para veres a tua jornada! 🌸',
      'tap_to_start': 'Toca para começar o desafio',
      'choose_level': 'Escolhe a tua fase de crescimento:',
      'view_journey': 'Ver Jornada de Crescimento',
      'progress': 'O Teu Progresso de Crescimento',
      'level_seedling': 'Plântula (Seedling)',
      'level_sprout': 'Broto (Sprout)',
      'level_leaf': 'Folha (Leaf)',
      'level_stem': 'Caule (Stem)',
      'level_bloom': 'Desabrochar (Bloom)',
      // Profile Screen
      'account': 'Conta',
      'display_name': 'Nome de Exibição',
      'save_name': 'Guardar Nome',
      'app_theme': 'Tema do Aplicativo',
      'select_color': 'Seleciona a tua cor Bloom:',
      'light_mode': 'Claro',
      'dark_mode': 'Escuro',
      'language': 'Idioma',
      'logout': 'Sair',
      'profile_updated': 'Perfil atualizado!',
      'pick_theme_color': 'Escolhe uma cor para o tema',
      'done': 'Concluído',
      'reset_all_progress': 'Redefinir Todo o Progresso',
      'reset_confirm_title': 'Tens a certeza?',
      'reset_confirm_message':
          'Isto irá apagar permanentemente a tua pontuação de confiança, série e todo o histórico. Esta ação não pode ser desfeita.',
      'cancel': 'Cancelar',
      'reset_everything': 'Redefinir Tudo',
      'reset_success': 'Todo o progresso foi redefinido.',
      // Shop Screen
      'shop_title': 'Loja Bloom',
      'your_points': 'Os Teus Pontos',
      'available_items': 'Itens Disponíveis',
      'streak_freeze': 'Congelar Série',
      'protects_streak': 'Protege a tua série de ser redefinida',
      'your_inventory': 'O Teu Inventário',
      'owned': 'Adquirido',
      'equipped': 'Equipado',
      'equip_freeze': 'Equipar Congelamento',
      'buy': 'Comprar',
      'freeze_purchased': 'Congelamento de Série Adquirido!',
      'not_enough_points': 'Pontos insuficientes!',
      'freeze_equipped': 'Congelamento equipado!',
      // Milestone Screen
      'your_growth_path': 'O Teu Caminho de Crescimento',
      'the_awakening': 'O Despertar',
      'seed_badge': 'Crachá de Semente',
      'first_spark': 'Primeira Faísca',
      'bronze_leaf': 'Folha de Bronze',
      'social_courage': 'Coragem Social',
      'silver_sprout': 'Broto de Prata',
      'confidence_bloom': 'Confiança em Flor',
      'gold_flower': 'Flor de Ouro',
      'mastery': 'Maestria',
      'diamond_crown': 'Coroa de Diamante',
      'milestone_claimed': 'Resgatado',
      'milestone_need_score': 'Necessita de {0}%',
      'milestone_claim_reward': 'Resgatar +{0} pts',
      'milestone_reward_toast': 'Resgatado! +{0} pts',
      // FAQ Screen
      'help_faq': 'Ajuda & FAQs',
      'common_questions': 'Perguntas Frequentes',
      'keep_blooming': 'Continua a Desabrochar! 🌸',
      'faq_q1': 'O que é o Bloom?',
      'faq_a1':
          'O Bloom é uma ferramenta de autoajuda desenvolvida para ajudar as pessoas a reduzir a ansiedade social através de um processo chamado \'Exposição Gradual\'. Ao completares pequenas e geríveis tarefas sociais, treinas o teu cérebro para perceber que as interações sociais são seguras e controláveis.',
      'faq_q2': 'Como funcionam os níveis?',
      'faq_a2':
          'Começamos com \'Plântula\' (tarefas muito fáceis) e avançamos até \'Desabrochar\' (tarefas mais desafiantes). À medida que completas as tarefas, ganhas pontos de confiança. Quanto maior for o nível, mais pontos ganhas!',
      'faq_q3': 'O que é o Medidor de Confiança?',
      'faq_a3':
          'A barra de progresso no teu ecrã inicial representa a tua confiança geral. Ela cresce à medida que completas tarefas. Atenção: se parares de praticar por vários dias, a tua pontuação de confiança pode diminuir ligeiramente, lembrando-te de que a confiança é um músculo que precisa de exercício regular!',
      'faq_q4': 'O que é uma Série (Streak)?',
      'faq_a4':
          'Uma série é a contagem de quantos dias consecutivos completaste pelo menos uma tarefa. A consistência é a chave para superar a ansiedade, por isso tenta manter a tua chama acesa!',
      'faq_q5': 'Onde são armazenados os meus dados?',
      'faq_a5':
          'A tua privacidade é a nossa prioridade. Todo o teu progresso, histórico e dados de perfil são armazenados localmente no teu próprio dispositivo. Nada é enviado para um servidor na nuvem.',
      'faq_q6': 'O que faço se o aplicativo falhar?',
      'faq_a6':
          'Se o aplicativo se comportar de forma estranha, tenta reiniciar o teu telemóvel. Se atualizaste o aplicativo, poderás precisar de limpar a cache do aplicativo nas definições do teu Android. Se tudo falhar, podes usar a opção \'Redefinir Todo o Progresso\' no teu Perfil.',
      'faq_q7': 'Posso saltar níveis?',
      'faq_a7':
          'Sim! Embora recomendemos o caminho gradual, és livre de escolher qualquer nível no mapa que consideres adequado para o teu nível de conforto atual.',
      'faq_q8': 'E se a tarefa for muito difícil?',
      'faq_a8':
          'Embora recomendemos que tentes realizar a tarefa, podes simplesmente voltar ao ecrã inicial e entrar novamente para mudar a tarefa atual.',
      'faq_q9': 'Contacta-nos',
      'faq_a9':
          'Gostaríamos muito de ouvir a opinião dos nossos utilizadores sobre o aplicativo e recomendações para atualizações futuras. Queremos saber quão bem o aplicativo funciona para ti, o que lhe falta e o que precisa de ser melhorado. Sente-te à vontade para partilhar o teu feedback em feedback.bloom@gmail.com, nós agradeceríamos imenso.',
      // Task Screen
      'stage_label': 'Fase: {0}',
      'keep_growing': 'Continua a crescer, {0}',
      'current_challenge': 'O teu desafio atual para esta fase:',
      'stage_mastered': 'Fase Dominada!',
      'all_done': 'Concluíste todos os desafios nesta fase.',
      'return_map': 'Voltar ao Mapa',
      'well_done': 'Bem Feito!',
      'i_completed': 'Eu Concluí Isto',
      'level_up_suggestion_title': 'Sugestão de subida de nível',
      'level_up_suggestion_message':
          'Você completou 10 tarefas em Este nível! Você está pronto para o próximo nível. Quer avançar?',
      'stay_here': 'Fique aqui',
      'move_to_next_level': 'Vá para o próximo nível',
      // Reflection Screen
      'reflect': 'Reflete sobre o teu crescimento',
      'challenge': 'Desafio',
      'anxiety_q': 'Quão ansioso te sentiste? (1-10)',
      'what_happened': 'O que realmente aconteceu?',
      'write_experience_hint': 'Escreve sobre a tua experiência...',
      'finish': 'Concluir Reflexão',
      // General / Auth
      'welcome': 'Bem-vindo ao Bloom',
      'subtitle': 'A space safe para fazeres crescer a tua confiança.',
      'start': 'Iniciar a Minha Jornada',
      'guest': 'Continuar como Convidado',
      'hello': 'Olá',
      'profile': 'Meu Perfil',
      'history': 'Minha Jornada de Crescimento',
      'streak': 'Série Atual',
      'best': 'Melhor Série',
      'points': 'Pontos de Confiança',
      //Splash Screen
      'loading': 'A carregar o teu jardim...',
    },
    'no': {
      // Progress Screen
      'your_journey': 'Din reise',
      'keep_growing_sub': 'Hvert lille skritt er en seier. Fortsett å vokse!',
      'how_it_works': 'Hvordan fungerer det?',
      'total_points': 'Totalt antall poeng',
      'current_streak': 'Gjeldende rekke',
      'tasks_done': 'Fullførte oppgaver',
      'rank': 'Rangering',
      'days': 'Dager',
      'contact_us': 'Kontakt oss',
      'contact_email_prompt':
          'For støtte og tilbakemeldinger, send oss en e-post på:',
      'close': 'Lukk',
      // Level Map Screen
      'tap_to_view_journey': 'Trykk for å se reisen din! 🌸',
      'tap_to_start': 'Trykk for å starte utfordringen',
      'choose_level': 'Velg ditt vekststadium:',
      'view_journey': 'Vis vekstreise',
      'progress': 'Din vekstfremgang',
      'level_seedling': 'Spire (Seedling)',
      'level_sprout': 'Spire (Sprout)',
      'level_leaf': 'Blad (Leaf)',
      'level_stem': 'Stilke (Stem)',
      'level_bloom': 'Blomstring (Bloom)',
      // Profile Screen
      'account': 'Konto',
      'display_name': 'Visningsnavn',
      'save_name': 'Lagre navn',
      'app_theme': 'App-tema',
      'select_color': 'Velg din Bloom-farge:',
      'light_mode': 'Lys',
      'dark_mode': 'Mørk',
      'language': 'Språk',
      'logout': 'Logg ut',
      'profile_updated': 'Profilen er oppdatert!',
      'pick_theme_color': 'Velg en temafarge',
      'done': 'Ferdig',
      'reset_all_progress': 'Nullstill all fremgang',
      'reset_confirm_title': 'Er du sikker?',
      'reset_confirm_message':
          'Dette vil slette selvtillitspoengene dine, rekken din og all historikk permanent. Dette kan ikke angres.',
      'cancel': 'Avbryt',
      'reset_everything': 'Nullstill alt',
      'reset_success': 'All fremgang har blitt nullstilt.',
      // Shop Screen
      'shop_title': 'Bloom-butikk',
      'your_points': 'Dine poeng',
      'available_items': 'Tilgjengelige gjenstander',
      'streak_freeze': 'Flamme-frys',
      'protects_streak': 'Beskytter rekken din mot å bli nullstilt',
      'your_inventory': 'Dine gjenstander',
      'owned': 'Eid',
      'equipped': 'Aktivert',
      'equip_freeze': 'Aktiver frys',
      'buy': 'Kjøp',
      'freeze_purchased': 'Flamme-frys kjøpt!',
      'not_enough_points': 'Ikke nok poeng!',
      'freeze_equipped': 'Frys aktivert!',
      // Milestone Screen
      'your_growth_path': 'Din vekststi',
      'the_awakening': 'Oppvåkningen',
      'seed_badge': 'Frø-merke',
      'first_spark': 'Første gnist',
      'bronze_leaf': 'Bronseblad',
      'social_courage': 'Sosialt mot',
      'silver_sprout': 'Sølvspire',
      'confidence_bloom': 'Selvtillitsblomstring',
      'gold_flower': 'Gullblomst',
      'mastery': 'Mestring',
      'diamond_crown': 'Diamantkrone',
      'milestone_claimed': 'Hentet',
      'milestone_need_score': 'Trenger {0}%',
      'milestone_claim_reward': 'Hent +{0} poeng',
      'milestone_reward_toast': 'Hentet! +{0} poeng',
      // FAQ Screen
      'help_faq': 'Hjelp og vanlige spørsmål',
      'common_questions': 'Vanlige spørsmål',
      'keep_blooming': 'Fortsett å blomstre! 🌸',
      'faq_q1': 'Hva er Bloom?',
      'faq_a1':
          'Bloom er et selvhjelpsverktøy designet for å hjelpe mennesker med å redusere sosial angst gjennom en prosess som kalles \'Gradvis eksponering\'. Ved å fullføre små, overkommelige sosiale oppgaver, trener du hjernen din til å innse at sosiale interaksjoner er trygge og håndterbare.',
      'faq_q2': 'Hvordan fungerer nivåene?',
      'faq_a2':
          'Vi starter med \'Frøling\' (veldig enkle oppgaver) og beveger oss opp til \'Blomstring\' (mer utfordrende oppgaver). Etter hvert som du fullfører oppgaver, tjener du selvtillitspoeng. Jo høyere nivået er, desto flere poeng tjener du!',
      'faq_q3': 'Hva er selvtillitsmåleren?',
      'faq_a3':
          'Fremdriftslinjen på startskjermen representerer din generelle selvtillit. Den vokser etter hvert som du fullfører oppgaver. Vær forsiktig: Hvis du slutter å øve i flere dager, kan selvtillitspoengene dine synke noe, for å minne deg på at selvtillit er en muskel som trenger regelmessig trening!',
      'faq_q4': 'Hva er en rekke (Streak)?',
      'faq_a4':
          'En rekke er tellingen av hvor mange dager på rad du har fullført minst én oppgave. Konsistens er nøkkelen til å overvinne angst, så prøv å holde flammen din brennende!',
      'faq_q5': 'Hvor lagres dataene mine?',
      'faq_a5':
          'Ditt personvern er vår prioritet. All fremgang, historikk og profildata lagres lokalt på din egen enhet. Ingenting blir lastet opp til en skyserver.',
      'faq_q6': 'Hva gjør jeg hvis appen krasjer?',
      'faq_a6':
          'Hvis appen oppfører seg merkelig, kan du prøve å starte telefonen på nytt. Hvis du har oppdatert appen, må du kanskje tømme app-bufferen (cache) i Android-innstillingene dine. Hvis alt annet mislykkes, kan du bruke alternativet \'Nullstill all fremgang\' under profilen din.',
      'faq_q7': 'Kan jeg hoppe over nivåer?',
      'faq_a7':
          'Ja! Selv om vi anbefaler den gradvise stien, står du fritt til å velge hvilket som helst nivå fra kartet som føles passende for ditt nåværende komfortnivå.',
      'faq_q8': 'Hva om oppgaven er veldig vanskelig?',
      'faq_a8':
          'Selv om vi anbefaler at du prøver å fullføre oppgaven, kan du bare gå tilbake til startskjermen og gå inn på nytt for å endre den nåværende oppgaven.',
      'faq_q9': 'Kontakt oss',
      'faq_a9':
          'Vi vil gjerne høre hva brukerne våre synes om appen vår, og tar gjerne imot anbefalinger for fremtidige oppdateringer. Vi vil gjerne høre hvor godt appen fungerer for deg, hva den mangler og hva som krever forbedring. Del gjerne tilbakemeldinger på feedback.bloom@gmail.com, det vil vi sette stor pris på.',
      // Task Screen
      'stage_label': 'Etappe: {0}',
      'keep_growing': 'Fortsett å vokse, {0}',
      'current_challenge': 'Din nåværende utfordring for denne etappen:',
      'stage_mastered': 'Etappe mestret!',
      'all_done': 'Du har fullført alle utfordringene i denne etappen.',
      'return_map': 'Tilbake til kartet',
      'well_done': 'Godt gjort!',
      'i_completed': 'Jeg har fullført dette',
      'level_up_suggestion_title': 'Level Up-forslag',
      'level_up_suggestion_message':
          'Du har fullført 10 oppgaver i Dette nivået! Du er klar for neste nivå. Vil du avansere?',
      'stay_here': 'Bo her',
      'move_to_next_level': 'Gå til neste nivå',
      // Reflection Screen
      'reflect': 'Reflekter over veksten din',
      'challenge': 'Utfordring',
      'anxiety_q': 'Hvor engstelig følte du deg? (1-10)',
      'what_happened': 'Hva skjedde egentlig?',
      'write_experience_hint': 'Skriv om opplevelsen din...',
      'finish': 'Fullfør refleksjon',
      // General / Auth
      'welcome': 'Velkommen til Bloom',
      'subtitle': 'Et trygt sted å dyrke selvtilliten din.',
      'start': 'Start min reise',
      'guest': 'Fortsett som gjest',
      'hello': 'Hei',
      'profile': 'Min profil',
      'history': 'Min vekstreise',
      'streak': 'Gjeldende rekke',
      'best': 'Beste rekke',
      'points': 'Selvtillitspoeng',
      //Splash Screen
      'loading': 'Laster hagen din...',
    },
    'gu': {
      // Progress Screen
      'your_journey': 'તમારી સફર',
      'keep_growing_sub': 'દરેક નાનું પગલું એક વિજય છે. આગળ વધતા રહો!',
      'how_it_works': 'તે કેવી રીતે કામ કરે છે?',
      'total_points': 'કુલ પોઇન્ટ્સ',
      'current_streak': 'વર્તમાન સાતત્ય (Streak)',
      'tasks_done': 'પૂર્ણ કરેલા કાર્યો',
      'rank': 'રેન્ક',
      'days': 'દિવસો',
      'contact_us': 'અમારો સંપર્ક કરો',
      'contact_email_prompt': 'સપોર્ટ અને ફીડબેક માટે, અમને અહીં ઈમેલ કરો:',
      'close': 'બંધ કરો',
      // Level Map Screen
      'tap_to_view_journey': 'તમારી સફર જોવા માટે ટેપ કરો! 🌸',
      'tap_to_start': 'ચેલેન્જ શરૂ કરવા માટે ટેપ કરો',
      'choose_level': 'તમારા વિકાસનો તબક્કો પસંદ કરો:',
      'view_journey': 'વિકાસની સફર જુઓ',
      'progress': 'તમારા વિકાસની પ્રગતિ',
      'level_seedling': 'છોડવા (Seedling)',
      'level_sprout': 'અંકુર (Sprout)',
      'level_leaf': 'પાંદડું (Leaf)',
      'level_stem': 'દાંડી (Stem)',
      'level_bloom': 'ખીલવું (Bloom)',
      // Profile Screen
      'account': 'ખાતું',
      'display_name': 'દર્શાવવાનું નામ',
      'save_name': 'નામ સેવ કરો',
      'app_theme': 'એપ થીમ',
      'select_color': 'તમારો બ્લૂમ રંગ પસંદ કરો:',
      'light_mode': 'લાઇટ',
      'dark_mode': 'ડાર્ક',
      'language': 'ભાષા',
      'logout': 'લોગઆઉట్',
      'profile_updated': 'પ્રોફાઇલ અપડેટ થઈ ગઈ!',
      'pick_theme_color': 'થીમનો રંગ પસંદ કરો',
      'done': 'પૂર્ણ',
      'reset_all_progress': 'બધી પ્રગતિ રીસેટ કરો',
      'reset_confirm_title': 'શું તમે ચોક્કસ છો?',
      'reset_confirm_message':
          'આ તમારા આત્મવિશ્વાસનો સ્કોર, સાતત્ય (Streak) અને તમામ ઇતિહાસ કાયમ માટે કાઢી નાખશે. આ પ્રક્રિયા પાછી ખેંચી શકાશે નહીં.',
      'cancel': 'રદ કરો',
      'reset_everything': 'બધું જ રીસેટ કરો',
      'reset_success': 'તમામ પ્રગતિ રીસેટ કરવામાં આવી છે.',
      // Shop Screen
      'shop_title': 'બ્લૂમ શોપ',
      'your_points': 'તમારા પોઇન્ટ્સ',
      'available_items': 'ઉપલબ્ધ વસ્તુઓ',
      'streak_freeze': 'સ્ટ્રીક ફ્રીઝ',
      'protects_streak': 'સાતત્ય (Streak) રીસેટ થતા અટકાવે છે',
      'your_inventory': 'તમારી વસ્તુઓ (Inventory)',
      'owned': 'ખરીદેલું',
      'equipped': 'વપરાશમાં છે',
      'equip_freeze': 'ફ્રીઝનો ઉપયોગ કરો',
      'buy': 'ખરીદો',
      'freeze_purchased': 'ફ્રીઝ ખરીદવામાં આવ્યું!',
      'not_enough_points': 'અપૂરતા પોઇન્ટ્સ છે!',
      'freeze_equipped': 'ફ્રીઝ એક્ટિવેટ થઈ ગયું છે!',
      // Milestone Screen
      'your_growth_path': 'તમારો વિકાસનો માર્ગ',
      'the_awakening': 'જાગૃતિ',
      'seed_badge': 'બીજ બેજ',
      'first_spark': 'પહેલો તણખો',
      'bronze_leaf': 'કાંસ્ય પર્ણ',
      'social_courage': 'સામાજિક હિંમત',
      'silver_sprout': 'રૂપેરી અંકુર',
      'confidence_bloom': 'આત્મવિશ્વાસનું ખીલવું',
      'gold_flower': 'સુવર્ણ પુષ્પ',
      'mastery': 'કુશળતા',
      'diamond_crown': 'હીરાનો મુગટ',
      'milestone_claimed': 'મળી ગયું',
      'milestone_need_score': '{0}% ની જરૂર છે',
      'milestone_claim_reward': 'ઇનામ મેળવો +{0} pts',
      'milestone_reward_toast': 'ઇનામ મળી ગયું! +{0} pts',
      // FAQ Screen
      'help_faq': 'મદદ અને FAQs',
      'common_questions': 'વારંવાર પૂછાતા પ્રશ્નો',
      'keep_blooming': 'ખીલતા રહો! 🌸',
      'faq_q1': 'બ્લૂમ (Bloom) શું છે?',
      'faq_a1':
          'બ્લૂમ એ એક સેલ્ફ-હેલ્પ ટૂલ છે જે \'ગ્રેડેડ એક્સપોઝર\' નામની પ્રક્રિયા દ્વારા લોકોની સામાજિક ચિંતા (Social Anxiety) ઘટાડવામાં મદદ કરવા માટે ડિઝાઇન કરવામાં આવ્યું છે. નાના, સરળ સામાજિક કાર્યો પૂર્ણ કરીને, તમે તમારા મગજને એ સમજવા માટે તાલીમ આપો છો કે સામાજિક વ્યવહારો સુરક્ષિત અને સામાન્ય છે.',
      'faq_q2': 'લેવલ્સ કેવી રીતે કામ કરે છે?',
      'faq_a2':
          'આપણે \'છોડવા\' (ખૂબ જ સરળ કાર્યો) થી શરૂઆત કરીએ છીએ અને ધીમે ધીમે \'ખીલવું\' (વધુ પડકારજનક કાર્યો) તરફ આગળ વધીએ છીએ. જેમ જેમ તમે કાર્યો પૂર્ણ કરશો, તેમ તેમ તમે આત્મવિશ્વાસના પોઇન્ટ્સ મેળવશો. લેવલ જેટલું ઊંચું હશે, તેટલા વધુ પોઇન્ટ્સ મળશે!',
      'faq_q3': 'કોન્ફિડન્સ મીટર શું છે?',
      'faq_a3':
          'તમારી હોમ સ્ક્રીન પરનો પ્રોગ્રેસ બાર તમારા એકંદર આત્મવિશ્વાસને દર્શાવે છે. જેમ જેમ તમે કાર્યો પૂર્ણ કરો છો તેમ તેમ તે વધે છે. સાવધ રહો: જો તમે થોડા દિવસો માટે પ્રેક્ટિસ કરવાનું બંધ કરી દો છો, તો તમારો આત્મવિશ્વાસ સ્કોર થોડો ઘટી શકે છે, જે તમને યાદ અપાવે છે કે આત્મવિશ્વાસ એ એક સ્નાયુ જેવો છે જેને નિયમિત કસરતની જરૂર હોય છે!',
      'faq_q4': 'સાતત્ય અથવા સ્ટ્રીક (Streak) શું છે?',
      'faq_a4':
          'સાતત્ય એ ગણતરી છે કે તમે સતત કેટલા દિવસો સુધી ઓછામાં ઓછું એક કાર્ય પૂર્ણ કર્યું છે. ચિંતા પર વિજય મેળવવા માટે સાતત્ય જ મુખ્ય ચાવી છે, તેથી તમારા ઉત્સાહની જ્યોતને પ્રજ્વલિત રાખવાનો પ્રયાસ કરો!',
      'faq_q5': 'મારો ડેટા ક્યાં સ્ટોર થાય છે?',
      'faq_a5':
          'તમારી પ્રાઇવસી અમારી પ્રાથમિકતા છે. તમારી બધી પ્રગતિ, ઇતિહાસ અને પ્રોફાઇલ ડેટા તમારા પોતાના ઉપકરણ પર જ સ્થાનિક રીતે (Locally) સ્ટોર થાય છે. ક્લાઉડ સર્વર પર કંઈપણ અપલોડ કરવામાં આવતું નથી.',
      'faq_q6': 'જો એપ ક્રેશ થાય તો મારે શું કરવું?',
      'faq_a6':
          'જો એપ વિચિત્ર રીતે કામ કરતી હોય, તો તમારા ફોનને રીસ્ટાર્ટ કરવાનો પ્રયાસ કરો. જો તમે એપ અપડેટ કરી હોય, તો તમારે તમારા એન્ડ્રોઇડ સેટિંગ્સમાં એપ કેશ (Cache) ક્લિયર કરવાની જરૂર પડી શકે છે. જો કંઈ કામ ન કરે, તો તમે તમારી પ્રોફાઇલમાં રહેલા \'બધી પ્રગતિ રીસેટ કરો\' વિકલ્પનો ઉપયોગ કરી શકો છો.',
      'faq_q7': 'શું હું લેવલ્સ સ્કીપ (નજરઅંદાજ) કરી શકું છું?',
      'faq_a7':
          'હા! જો કે અમે ક્રમિક માર્ગની ભલામણ કરીએ છીએ, પણ તમે તમારા વર્તમાન કમ્ફર્ટ લેવલ અનુસાર મેપમાંથી કોઈ પણ લેવલ પસંદ કરવા માટે સંપૂર્ણ સ્વતંત્ર છો.',
      'faq_q8': 'જો કાર્ય ખૂબ જ અઘરું હોય તો શું કરવું?',
      'faq_a8':
          'જો કે અમે તમને કાર્ય પૂર્ણ કરવાનો પ્રયાસ કરવાની ભલામણ કરીએ છીએ, તેમ છતાં જો તે ખૂબ મુશ્કેલ હોય, તો તમે ફક્ત હોમ સ્ક્રીન પર પાછા જઈ શકો છો, અને વર્તમાન કાર્ય બદલવા માટે ફરીથી પ્રવેશી શકો છો.',
      'faq_q9': 'અમારો સંપર્ક કરો',
      'faq_a9':
          'અમે અમારા વપરાશકર્તાઓ પાસેથી એપ વિશેના તેમના અભિપ્રાયો અને ભવિષ્યના અપડેટ્સ માટેના સૂચનો જાણવા ઈચ્છીએ છીએ. આ એપ વપરાશકર્તાઓ માટે કેટલી સારી રીતે કામ કરી રહી છે, તેમાં શેની કમી છે અને ક્યાં સુધારાની જરૂર છે તે અમને જાણવું ગમશે. કૃપા કરીને feedback.bloom@gmail.com પર તમારો ફીડબેક શેર કરવામાં સંકોચ ન કરશો, અમે તેની ખરેખर પ્રશંસા કરીશું.',
      // Task Screen
      'stage_label': 'તબક્કો: {0}',
      'keep_growing': 'આગળ વધતા રહો, {0}',
      'current_challenge': 'આ તબક્કા માટે તમારો વર્તમાન પડકાર:',
      'stage_mastered': 'તબક્કો સફળતાપૂર્વક પાર પાડ્યો!',
      'all_done': 'તમે આ તબક્કાના તમામ પડકારો પૂર્ણ કર્યા છે.',
      'return_map': 'મેપ પર પાછા જાઓ',
      'well_done': 'ખૂબ સરસ!',
      'i_completed': 'મેં આ પૂર્ણ કર્યું છે',
      'level_up_suggestion_title': 'લેવલ અપ માટે સૂચન',
      'level_up_suggestion_message':
          'તમે આ લેવલમાં ૧૦ કાર્યો પૂર્ણ કર્યા છે! તમે આગલા લેવલ માટે તૈયાર છો. શું તમે આગળ વધવા માંગો છો?',
      'stay_here': 'અહીં જ રહો',
      'move_to_next_level': 'આગલા લેવલ પર જાઓ',
      // Reflection Screen
      'reflect': 'તમારા વિકાસ પર પ્રકાશ પાડો',
      'challenge': 'પડકાર',
      'anxiety_q': 'તમે કેટલા ચિંતિત હતા? (1-10)',
      'what_happened': 'વાસ્તવમાં શું બન્યું હતું?',
      'write_experience_hint': 'તમારા અનુભવ વિશે લખો...',
      'finish': 'સમીક્ષા પૂર્ણ કરો',
      // General / Auth
      'welcome': 'બ્લૂમમાં આપનું સ્વાગત છે',
      'subtitle': 'તમારો આત્મવિश्वાસ વધારવા માટેનું એક સુરક્ષિત સ્થાન.',
      'start': 'મારી સફર શરૂ કરો',
      'guest': 'મહેમાન તરીકે આગળ વધો',
      'hello': 'નમસ્તે',
      'profile': 'મારી પ્રોફાઇલ',
      'history': 'મારા વિકાસની સફર',
      'streak': 'વર્તમાન સાતત્ય',
      'best': 'શ્રેષ્ઠ સાતત્ય',
      'points': 'આત્મવિશ્વાસ પોઇન્ટ્સ',
      //Splash Screen
      'loading': 'તમારો બગીચો લોડ થઈ રહ્યો છે...',
    },
    'pa': {
      // Progress Screen
      'your_journey': 'ਤੁਹਾਡਾ ਸਫ਼ਰ',
      'keep_growing_sub': 'ਹਰ ਛੋਟਾ ਕਦਮ ਇੱਕ ਜਿੱਤ ਹੈ। ਅੱਗੇ ਵਧਦੇ ਰਹੋ!',
      'how_it_works': 'ਇਹ ਕਿਵੇਂ ਕੰਮ ਕਰਦਾ ਹੈ?',
      'total_points': 'ਕੁੱਲ ਅੰਕ',
      'current_streak': 'ਮੌਜੂਦਾ ਸਿਲਸਿਲਾ (Streak)',
      'tasks_done': 'ਪੂਰੇ ਕੀਤੇ ਕੰਮ',
      'rank': 'ਰੈਂਕ',
      'days': 'ਦਿਨ',
      'contact_us': 'ਸਾਡੇ ਨਾਲ ਸੰਪਰਕ ਕਰੋ',
      'contact_email_prompt': 'ਸਹਾਇਤਾ ਅਤੇ ਫੀਡਬੈਕ ਲਈ, ਸਾਨੂੰ ਇੱਥੇ ਈਮੇਲ ਕਰੋ:',
      'close': 'ਬੰਦ ਕਰੋ',
      // Level Map Screen
      'tap_to_view_journey': 'ਆਪਣਾ ਸਫ਼ਰ ਦੇਖਣ ਲਈ ਟੈਪ ਕਰੋ! 🌸',
      'tap_to_start': 'ਚੈਲੇਂਜ ਸ਼ੁਰੂ ਕਰਨ ਲਈ ਟੈਪ ਕਰੋ',
      'choose_level': 'ਆਪਣੇ ਵਿਕਾਸ ਦਾ ਪੜਾਅ ਚੁਣੋ:',
      'view_journey': 'ਵਿਕਾਸ ਦਾ ਸਫ਼ਰ ਦੇਖੋ',
      'progress': 'ਤੁਹਾਡੇ ਵਿਕਾਸ ਦੀ ਪ੍ਰਗਤੀ',
      'level_seedling': 'ਪੌਦਾ (Seedling)',
      'level_sprout': 'ਅੰਕੁਰ (Sprout)',
      'level_leaf': 'ਪੱਤਾ (Leaf)',
      'level_stem': 'ਡੰਡੀ (Stem)',
      'level_bloom': 'ਖਿੜਨਾ (Bloom)',
      // Profile Screen
      'account': 'ਖਾਤਾ',
      'display_name': 'ਦਿਖਾਉਣ ਵਾਲਾ ਨਾਮ',
      'save_name': 'ਨਾਮ ਸੇਵ ਕਰੋ',
      'app_theme': 'ਐਪ ਥੀਮ',
      'select_color': 'ਆਪਣਾ ਬਲੂਮ ਰੰਗ ਚੁਣੋ:',
      'light_mode': 'ਲਾਈਟ',
      'dark_mode': 'ਡਾਰਕ',
      'language': 'ਭਾਸ਼ਾ',
      'logout': 'ਲੌਗਆਉਟ',
      'profile_updated': 'ਪ੍ਰੋਫਾਈਲ ਅਪਡੇਟ ਹੋ ਗਈ!',
      'pick_theme_color': 'ਥੀਮ ਦਾ ਰੰਗ ਚੁਣੋ',
      'done': 'ਪੂਰਾ',
      'reset_all_progress': 'ਸਾਰੀ ਪ੍ਰਗਤੀ ਰੀਸੈਟ ਕਰੋ',
      'reset_confirm_title': 'ਕੀ ਤੁਹਾਨੂੰ ਪੱਕਾ ਯਕੀਨ ਹੈ?',
      'reset_confirm_message':
          'ਇਹ ਤੁਹਾਡੇ ਆਤਮ-ਵਿਸ਼ਵਾਸ ਦਾ ਸਕੋਰ, ਸਿਲਸਿਲਾ (Streak) ਅਤੇ ਸਾਰਾ ਇਤਿਹਾਸ ਹਮੇਸ਼ਾ ਲਈ ਮਿਟਾ ਦੇਵੇਗਾ। ਇਹ ਪ੍ਰਕਿਰਿਆ ਵਾਪਸ ਨਹੀਂ ਲਈ ਜਾ ਸਕਦੀ।',
      'cancel': 'ਰੱਦ ਕਰੋ',
      'reset_everything': 'ਸਭ ਕੁਝ ਰੀਸੈਟ ਕਰੋ',
      'reset_success': 'ਸਾਰੀ ਪ੍ਰਗਤੀ ਰੀਸੈਟ ਕਰ ਦਿੱਤੀ ਗਈ ਹੈ।',
      // Shop Screen
      'shop_title': 'ਬਲੂਮ ਸ਼ਾਪ',
      'your_points': 'ਤੁਹਾਡੇ ਅੰਕ',
      'available_items': 'ਉਪਲਬਧ ਚੀਜ਼ਾਂ',
      'streak_freeze': 'ਸਿਲਸਿਲਾ ਫ੍ਰੀਜ਼ (Streak Freeze)',
      'protects_streak': 'ਸਿਲਸਿਲੇ (Streak) ਨੂੰ ਰੀਸੈਟ ਹੋਣ ਤੋਂ ਰੋਕਦਾ ਹੈ',
      'your_inventory': 'ਤੁਹਾਡੀਆਂ ਚੀਜ਼ਾਂ (Inventory)',
      'owned': 'ਖਰੀਦਿਆ ਹੋਇਆ',
      'equipped': 'ਵਰਤੋਂ ਵਿੱਚ ਹੈ',
      'equip_freeze': 'ਫ੍ਰੀਜ਼ ਦੀ ਵਰਤੋਂ ਕਰੋ',
      'buy': 'ਖਰੀਦੋ',
      'freeze_purchased': 'ਫ੍ਰੀਜ਼ ਖਰੀਦ ਲਿਆ ਗਿਆ ਹੈ!',
      'not_enough_points': 'ਲੋੜੀਂਦੇ ਅੰਕ ਨਹੀਂ ਹਨ!',
      'freeze_equipped': 'ਫ੍ਰੀਜ਼ ਐਕਟੀਵੇਟ ਹੋ ਗਿਆ ਹੈ!',
      // Milestone Screen
      'your_growth_path': 'ਤੁਹਾਡਾ ਵਿਕਾਸ ਮਾਰਗ',
      'the_awakening': 'ਜਾਗ੍ਰਿਤੀ',
      'seed_badge': 'ਬੀਜ ਬੈਜ',
      'first_spark': 'ਪਹਿਲੀ ਚੰਗਿਆੜੀ',
      'bronze_leaf': 'ਕਾਂਸੀ ਦਾ ਪੱਤਾ',
      'social_courage': 'ਸਮਾਜਿਕ ਹਿੰਮਤ',
      'silver_sprout': 'ਚਾਂਦੀ ਦਾ ਅੰਕੁਰ',
      'confidence_bloom': 'ਆਤਮ-ਵਿਸ਼ਵਾਸ ਦਾ ਖਿੜਨਾ',
      'gold_flower': 'ਸੁਨਹਿਰੀ ਫੁੱਲ',
      'mastery': 'ਮਾਹਰਤਾ',
      'diamond_crown': 'ਹੀਰੇ ਦਾ ਤਾਜ',
      'milestone_claimed': 'ਪ੍ਰਾਪਤ ਕੀਤਾ',
      'milestone_need_score': '{0}% ਦੀ ਲੋੜ ਹੈ',
      'milestone_claim_reward': 'ਇਨਾਮ ਲਓ +{0} ਅੰਕ',
      'milestone_reward_toast': 'ਇਨਾਮ ਮਿਲ ਗਿਆ! +{0} ਅੰਕ',
      // FAQ Screen
      'help_faq': 'ਮਦਦ ਅਤੇ FAQs',
      'common_questions': 'ਅਕਸਰ ਪੁੱਛੇ ਜਾਣ ਵਾਲੇ ਸਵਾਲ',
      'keep_blooming': 'ਖਿੜਦੇ ਰਹੋ! 🌸',
      'faq_q1': 'ਬਲੂਮ (Bloom) ਕੀ ਹੈ?',
      'faq_a1':
          'ਬਲੂਮ ਇੱਕ ਸੈਲਫ-ਹੈਲਪ ਟੂਲ ਹੈ ਜੋ \'ਗ੍ਰੈਡੇਡ ਐਕਸਪੋਜ਼ਰ\' ਨਾਮਕ ਪ੍ਰਕਿਰਿਆ ਰਾਹੀਂ ਲੋਕਾਂ ਦੀ ਸਮਾਜਿਕ ਚਿੰਤਾ (Social Anxiety) ਨੂੰ ਘਟਾਉਣ ਵਿੱਚ ਮਦਦ ਕਰਨ ਲਈ ਡਿਜ਼ਾਈਨ ਕੀਤਾ ਗਿਆ ਹੈ। ਛੋਟੇ, ਸੌਖੇ ਸਮਾਜਿਕ ਕੰਮਾਂ ਨੂੰ ਪੂਰਾ ਕਰਕੇ, ਤੁਸੀਂ ਆਪਣੇ ਦਿਮਾਗ ਨੂੰ ਇਹ ਸਮਝਣ ਦੀ ਸਿਖਲਾਈ ਦਿੰਦੇ ਹੋ ਕਿ ਸਮਾਜਿਕ ਗੱਲਬਾਤ ਸੁਰੱਖਿਅਤ ਅਤੇ ਆਮ ਹੈ।',
      'faq_q2': 'ਲੈਵਲ ਕਿਵੇਂ ਕੰਮ ਕਰਦੇ ਹਨ?',
      'faq_a2':
          'ਅਸੀਂ \'ਪੌਦਾ\' (ਬਹੁਤ ਹੀ ਸੌਖੇ ਕੰਮ) ਤੋਂ ਸ਼ੁਰੂਆਤ ਕਰਦੇ ਹਾਂ ਅਤੇ ਹੌਲੀ-ਹੌਲੀ \'ਖਿੜਨਾ\' (ਵਧੇਰੇ ਚੁਣੌਤੀਪੂਰਨ ਕੰਮ) ਵੱਲ ਵਧਦੇ ਹਾਂ। ਜਿਵੇਂ-ਜਿਵੇਂ ਤੁਸੀਂ ਕੰਮ ਪੂਰੇ ਕਰੋਗੇ, ਤੁਸੀਂ ਆਤਮ-ਵਿਸ਼ਵਾਸ ਦੇ ਅੰਕ ਪ੍ਰਾਪਤ ਕਰੋਗੇ। ਲੈਵਲ ਜਿੰਨਾ ਉੱਚਾ ਹੋਵੇਗਾ, ਉੱਨੇ ਹੀ ਜ਼ਿਆਦਾ ਅੰਕ ਮਿਲਣਗੇ!',
      'faq_q3': 'ਕਾਨਫੀਡੈਂਸ ਮੀਟਰ ਕੀ ਹੈ?',
      'faq_a3':
          'ਤੁਹਾਡੀ ਹੋਮ ਸਕ੍ਰੀਨ \'ਤੇ ਪ੍ਰੋਗ੍ਰੈਸ ਬਾਰ ਤੁਹਾਡੇ ਸਮੁੱਚੇ ਆਤਮ-ਵਿਸ਼ਵਾਸ ਨੂੰ ਦਰਸਾਉਂਦੀ ਹੈ। ਜਿਵੇਂ-ਜਿਵੇਂ ਤੁਸੀਂ ਕੰਮ ਪੂਰੇ ਕਰਦੇ ਹੋ, ਇਹ ਵਧਦੀ ਹੈ। ਸਾਵਧਾਨ ਰਹੋ: ਜੇਕਰ ਤੁਸੀਂ ਕੁਝ ਦਿਨਾਂ ਲਈ ਅਭਿਆਸ ਕਰਨਾ ਬੰਦ ਕਰ ਦਿੰਦੇ ਹੋ, ਤਾਂ ਤੁਹਾਡਾ ਆਤਮ-ਵਿਸ਼ਵਾਸ ਸਕੋਰ ਥੋੜ੍ਹਾ ਘਟ ਸਕਦਾ ਹੈ, ਜੋ ਤੁਹਾਨੂੰ ਯਾਦ ਦਿਵਾਉਂਦਾ ਹੈ ਕਿ ਆਤਮ-ਵਿਸ਼ਵਾਸ ਇੱਕ ਮਾਸਪੇਸ਼ੀ ਵਰਗਾ ਹੈ ਜਿਸ ਨੂੰ ਨਿਯਮਿਤ ਕਸਰਤ ਦੀ ਲੋੜ ਹੁੰਦੀ ਹੈ!',
      'faq_q4': 'ਸਿਲਸਿਲਾ ਜਾਂ ਸਟ੍ਰੀਕ (Streak) ਕੀ ਹੈ?',
      'faq_a4':
          'ਸਿਲਸਿਲਾ ਇਹ ਗਿਣਤੀ ਹੈ ਕਿ ਤੁਸੀਂ ਲਗਾਤਾਰ ਕਿੰਨੇ ਦਿਨਾਂ ਤੱਕ ਘੱਟੋ-ਘੱਟ ਇੱਕ ਕੰਮ ਪੂਰਾ ਕੀਤਾ ਹੈ। ਚਿੰਤਾ \'ਤੇ ਜਿੱਤ ਪਾਉਣ ਲਈ ਨਿਰੰਤਰਤਾ ਹੀ ਮੁੱਖ ਚਾਬੀ ਹੈ, ਇਸ ਲਈ ਆਪਣੇ ਉਤਸ਼ਾਹ ਦੀ ਲਾਟ ਨੂੰ ਜਗਾ ਕੇ ਰੱਖਣ ਦੀ ਕੋਸ਼ਿਸ਼ ਕਰੋ!',
      'faq_q5': 'ਮੇਰਾ ਡੇਟਾ ਕਿੱਥੇ ਸਟੋਰ ਹੁੰਦਾ ਹੈ?',
      'faq_a5':
          'ਤੁਹਾਡੀ ਪ੍ਰਾਈਵੇਸੀ ਸਾਡੀ ਪਹਿਲ ਹੈ। ਤੁਹਾਡੀ ਸਾਰੀ ਪ੍ਰਗਤੀ, ਇਤਿਹਾਸ ਅਤੇ ਪ੍ਰੋਫਾਈਲ ਡેટਾ ਤੁਹਾਡੀ ਆਪਣੀ ਡਿਵਾਈਸ \'ਤੇ ਹੀ ਲੋਕਲ ਤੌਰ \'ਤੇ (Locally) ਸਟੋਰ ਹੁੰਦਾ ਹੈ। ਕਲਾਊਡ ਸਰਵਰ \'ਤੇ ਕੁਝ ਵੀ ਅਪਲੋਡ ਨਹੀਂ ਕੀਤਾ ਜਾਂਦਾ।',
      'faq_q6': 'ਜੇਕਰ ਐਪ ਕ੍ਰੈਸ਼ ਹੁੰਦੀ ਹੈ ਤਾਂ ਮੈਨੂੰ ਕੀ ਕਰਨਾ ਚਾਹੀਦਾ ਹੈ?',
      'faq_a6':
          'ਜੇਕਰ ਐਪ ਅਜੀਬ ਤਰੀਕੇ ਨਾਲ ਕੰਮ ਕਰ ਰਹੀ ਹੈ, ਤਾਂ ਆਪਣੇ ਫੋਨ ਨੂੰ ਰੀਸਟਾਰਟ ਕਰਨ ਦੀ ਕੋਸ਼ਿਸ਼ ਕਰੋ। ਜੇਕਰ ਤੁਸੀਂ ਐਪ ਅਪਡੇਟ ਕੀਤੀ ਹੈ, ਤਾਂ ਤੁਹਾਨੂੰ ਆਪਣੇ ਐਂਡਰਾਇਡ ਸੈਟਿੰਗਜ਼ ਵਿੱਚ ਐਪ ਕੈਸ਼ (Cache) ਕਲੀਅਰ ਕਰਨ ਦੀ ਲੋੜ ਹੋ ਸਕਦੀ ਹੈ। ਜੇਕਰ ਕੁਝ ਵੀ ਕੰਮ ਨਾ ਕਰੇ, ਤਾਂ ਤੁਸੀਂ ਆਪਣੀ ਪ੍ਰੋਫਾਈਲ ਵਿੱਚ ਦਿੱਤੇ \'ਸਾਰੀ ਪ੍ਰਗਤੀ ਰੀਸੈਟ ਕਰੋ\' ਵਿਕਲਪ ਦੀ ਵਰਤੋਂ ਕਰ ਸਕਦੇ ਹੋ।',
      'faq_q7': 'ਕੀ ਮੈਂ ਲੈਵਲ ਸਕਿਪ (ਛੱਡ) ਸਕਦਾ ਹਾਂ?',
      'faq_a7':
          'ਹਾਂ! ਭਾਵੇਂ ਅਸੀਂ ਹੌਲੀ-ਹੌਲੀ ਅੱਗੇ ਵਧਣ ਦੇ ਰਸਤੇ ਦੀ ਸਿਫ਼ਾਰਸ਼ ਕਰਦੇ ਹਾਂ, ਪਰ ਤੁਸੀਂ ਆਪਣੇ ਮੌਜੂਦਾ ਕੰਫਰਟ ਲੈਵਲ ਅਨੁਸਾਰ ਮੈਪ ਵਿੱਚੋਂ ਕੋਈ ਵੀ ਲੈਵਲ ਚੁਣਨ ਲਈ ਪੂਰੀ ਤਰ੍ਹਾਂ ਸੁਤੰਤਰ ਹੋ।',
      'faq_q8': 'ਜੇਕਰ ਕੰਮ ਬਹੁਤ ਮੁਸ਼ਕਲ ਹੋਵੇ ਤਾਂ ਕੀ ਕਰਨਾ ਚਾਹੀਦਾ ਹੈ?',
      'faq_a8':
          'ਭਾਵੇਂ ਅਸੀਂ ਤੁਹਾਨੂੰ ਕੰਮ ਪੂਰਾ ਕਰਨ ਦੀ ਕੋਸ਼ਿਸ਼ ਕਰਨ ਦੀ ਸਿਫ਼ਾਰਸ਼ ਕਰਦੇ ਹਾਂ, ਫਿਰ ਵੀ ਜੇਕਰ ਇਹ ਬਹੁਤ ਮੁਸ਼ਕਲ ਹੋਵੇ, ਤਾਂ ਤੁਸੀਂ ਸਿਰਫ਼ ਹੋਮ ਸਕ੍ਰੀਨ \'ਤੇ ਵਾਪਸ ਜਾ ਸਕਦੇ ਹੋ, ਅਤੇ ਮੌਜੂਦਾ ਕੰਮ ਬਦਲਣ ਲਈ ਦੁਬਾਰਾ ਐਂਟਰ ਕਰ ਸਕਦੇ ਹੋ।',
      'faq_q9': 'ਸਾਡੇ ਨਾਲ ਸੰਪਰਕ ਕਰੋ',
      'faq_a9':
          'ਸਾਨੂੰ ਸਾਡੇ ਯੂਜ਼ਰਸ ਤੋਂ ਐਪ ਬਾਰੇ ਉਨ੍ਹਾਂ ਦੇ ਵਿਚਾਰ ਅਤੇ ਭਵਿੱਖ ਦੇ ਅਪਡੇਟਸ ਲਈ ਸੁਝਾਅ ਜਾਣਨਾ ਬਹੁਤ ਪਸੰਦ ਆਵੇਗਾ। ਅਸੀਂ ਇਹ ਜਾਣਨਾ ਚਾਹੁੰਦੇ ਹਾਂ ਕਿ ਐਪ ਯੂਜ਼ਰਸ ਲਈ ਕਿੰਨੀ ਚੰਗੀ ਤਰ੍ਹਾਂ ਕੰਮ ਕਰ ਰਹੀ ਹੈ, ਇਸ ਵਿੱਚ ਕਿਸ ਚੀਜ਼ ਦੀ ਕਮੀ ਹੈ ਅਤੇ ਕਿੱਥੇ ਸੁਧਾਰ ਦੀ ਲੋੜ ਹੈ। ਕਿਰਪਾ ਕਰਕੇ feedback.bloom@gmail.com \'ਤੇ ਆਪਣਾ ਫੀਡਬੈਕ ਸਾਂਝਾ ਕਰਨ ਵਿੱਚ ਸੰਕੋਚ ਨਾ ਕਰੋ, ਅਸੀਂ ਇਸਦੀ ਸੱਚਮੁੱਚ ਬਹੁਤ ਸ਼ਲਾਘਾ ਕਰਾਂਗੇ।',
      // Task Screen
      'stage_label': 'ਪੜਾਅ: {0}',
      'keep_growing': 'ਅੱਗੇ ਵਧਦੇ ਰਹੋ, {0}',
      'current_challenge': 'ਇਸ ਪੜਾਅ ਲਈ ਤੁਹਾਡੀ ਮੌਜੂਦਾ ਚੁਣੌਤੀ:',
      'stage_mastered': 'ਪੜਾਅ ਸਫਲਤਾਪੂਰਵਕ ਪੂਰਾ ਹੋਇਆ!',
      'all_done': 'ਤੁਸੀਂ ਇਸ ਪੜਾਅ ਦੀਆਂ ਸਾਰੀਆਂ ਚੁਣੌਤੀਆਂ ਪੂਰੀਆਂ ਕਰ ਲਈਆਂ ਹਨ।',
      'return_map': 'ਮੈਪ \'ਤੇ ਵਾਪਸ ਜਾਓ',
      'well_done': 'ਬਹੁਤ ਵਧੀਆ!',
      'i_completed': 'ਮੈਂ ਇਹ ਪੂਰਾ ਕਰ ਲਿਆ ਹੈ',
      'level_up_suggestion_title': 'ਲੈਵਲ ਅਪ ਲਈ ਸੁਝਾਅ',
      'level_up_suggestion_message':
          'ਤੁਸੀਂ ਇਸ ਲੈਵਲ ਵਿੱਚ 10 ਕੰਮ ਪੂਰੇ ਕਰ ਲਏ ਹਨ! ਤੁਸੀਂ ਅਗਲੇ ਲੈਵਲ ਲਈ ਤਿਆਰ ਹੋ। ਕੀ ਤੁਸੀਂ ਅੱਗੇ ਵਧਣਾ ਚਾਹੁੰਦੇ ਹੋ?',
      'stay_here': 'ਇੱਥੇ ਹੀ ਰਹੋ',
      'move_to_next_level': 'ਅਗਲੇ ਲੈਵਲ \'ਤੇ ਜਾਓ',
      // Reflection Screen
      'reflect': 'ਆਪਣੇ ਵਿਕਾਸ ਬਾਰੇ ਸੋਚੋ',
      'challenge': 'ਚੁਣੌਤੀ',
      'anxiety_q': 'ਤੁਸੀਂ ਕਿੰਨੀ ਚਿੰਤਾ ਮਹਿਸੂਸ ਕੀਤੀ? (1-10)',
      'what_happened': 'ਅਸਲ ਵਿੱਚ ਕੀ ਹੋਇਆ ਸੀ?',
      'write_experience_hint': 'ਆਪਣੇ ਅਨੁਭਵ ਬਾਰੇ ਲਿਖੋ...',
      'finish': 'ਸਮੀਖਿਆ ਪੂਰੀ ਕਰੋ',
      // General / Auth
      'welcome': 'ਬਲੂਮ ਵਿੱਚ ਤੁਹਾਡਾ ਸੁਆਗਤ ਹੈ',
      'subtitle': 'ਤੁਹਾਡਾ ਆਤਮ-ਵਿਸ਼ਵਾਸ ਵਧਾਉਣ ਲਈ ਇੱਕ ਸੁਰੱਖਿਅਤ ਜਗ੍ਹਾ।',
      'start': 'ਮੇਰਾ ਸਫ਼ਰ ਸ਼ੁਰੂ ਕਰੋ',
      'guest': 'ਮਹਿਮਾਨ ਵਜੋਂ ਅੱਗੇ ਵਧੋ',
      'hello': 'ਸਤਿ ਸ਼੍ਰੀ ਅਕਾਲ',
      'profile': 'ਮੇਰੀ ਪ੍ਰੋਫਾਈਲ',
      'history': 'ਮੇਰੇ ਵਿਕਾਸ ਦਾ ਸਫ਼ਰ',
      'streak': 'ਮੌਜੂਦਾ ਸਿਲਸਿਲਾ',
      'best': 'ਸਭ ਤੋਂ ਵਧੀਆ ਸਿਲਸਿਲਾ',
      'points': 'ਆਤਮ-ਵਿਸ਼ਵਾਸ ਅੰਕ',
      //Splash Screen
      'loading': 'ਤੁਹਾਡਾ ਬਾਗ ਲੋਡ ਹੋ ਰਿਹਾ ਹੈ...',
    },
    'mni': {
      // Progress Screen
      'your_journey': 'নহা গী খোঙচৎ',
      'keep_growing_sub': 'খুদোল খুদিংমক মায় পাকপনি। মখা তানা চাওখৎলু!',
      'how_it_works': 'মসি করম্না থবক তৌবগে?',
      'total_points': 'அপুনবা প্লোইন্ট',
      'current_streak': 'হৌজিক লৈরিবা মখা তাবা নুমিত',
      'tasks_done': 'লোইশিনখ্রবা থবকশিং',
      'rank': 'র্যাঙ্ক',
      'days': 'নুমীৎ',
      'contact_us': 'ঐখোয়গা যোগাযোগ তৌবীয়ু',
      'contact_email_prompt': 'মচেৎ অমসুং সপোর্তকীদমক, ঐখোয়দা ইমেল তৌবীয়ু:',
      'close': 'থিংজিল্লু',
      // Level Map Screen
      'tap_to_view_journey': 'নহা গী খোঙচৎ য়েংনবা নাম্মু! 🌸',
      'tap_to_start': 'চ্যালেঞ্জ হৌনબા নাম্মু',
      'choose_level': 'নহা গী চাওখৎপগী তাঙ্কক খল্লু:',
      'view_journey': 'চাওখৎপগী খোঙচৎ য়েঙু',
      'progress': 'নহা গী চাওখৎপগী খুমাঙ চাউশিনবা',
      'level_seedling': 'মরূ (Seedling)',
      'level_sprout': 'ফোঙগৎলকপা (Sprout)',
      'level_leaf': 'উনা (Leaf)',
      'level_stem': 'মশা (Stem)',
      'level_bloom': 'সাতপা (Bloom)',
      // Profile Screen
      'account': 'একাউন্ট',
      'display_name': 'উৎকদবা মিং',
      'save_name': 'মিং সেভ তৌও',
      'app_theme': 'অ্যাপ থিম',
      'select_color': 'নহা গী ব্লুম মচু খল্লু:',
      'light_mode': 'মঙাল লৈবা',
      'dark_mode': 'অমুম্বা',
      'language': 'লোন',
      'logout': 'লোগ আউট তৌও',
      'profile_updated': 'પ્રોફાઇલ অপเดট তৌখ্রে!',
      'pick_theme_color': 'থিমগী মচু খল্লু',
      'done': 'লোইখ্রে',
      'reset_all_progress': 'খুমাঙ চাউশিনবা পুম্নমক অমুক হন্না সেৎ তৌও',
      'reset_confirm_title': 'নহাক চান্না থাজબ্রা?',
      'reset_confirm_message':
          'মসিনা নহা গী থাজজগী স্কোর, মখা তাবা নুমিত অমসুং পুম্নমক পুৱারী লেংදনা মুত্থৎকনি। মসি অমুক হন্না লৌথোকপা য়ারোই।',
      'cancel': 'বাতিল তৌও',
      'reset_everything': 'পুম্নমক রিসেট তৌও',
      'reset_success': 'ખુমাঙ চাউশিনবা পুম্নมক রিসেট তৌখ্রে।',
      // Shop Screen
      'shop_title': 'ব্লুম শপ',
      'your_points': 'নহা গী পোઇন্টশিং',
      'available_items': 'ফংলিব পোৎলমশিং',
      'streak_freeze': 'ফ্রীজ ফ্লেম (Streak Freeze)',
      'protects_streak': 'মখা তাবা নুমিতশিং অমুক হন্না সেৎ তৌবদগী ঙাকথোকই',
      'your_inventory': 'নহা গী বেগ (Inventory)',
      'owned': 'লৈখ্রবা',
      'equipped': 'শীজিন্নরিবা',
      'equip_freeze': 'ফ্রীজ শীজিন্নৌ',
      'buy': 'লৈয়ু',
      'freeze_purchased': 'ফ্রীজ লৈখ্রে!',
      'not_enough_points': 'পোਇন্ট মহু খাকতে!',
      'freeze_equipped': 'ফ্রীজ এক্টিভেট তৌখ্রে!',
      // Milestone Screen
      'your_growth_path': 'নহা গী চাওখৎপগী লম্বী',
      'the_awakening': 'হৌগৎপা',
      'seed_badge': 'মরূ ব্যাজ',
      'first_spark': 'অহানবা মঙাল',
      'bronze_leaf': 'ব্রোঞ্জ উના',
      'social_courage': 'মীয়াম মরক্তা থৌনা লৈবা',
      'silver_sprout': 'লুপা ফোঙগৎলকপা',
      'confidence_bloom': 'ਥাজজগী সাতপা',
      'gold_flower': 'সনাগী লৈ',
      'mastery': 'মায় পাকপা',
      'diamond_crown': 'হীরাগী লুগুপ',
      'milestone_claimed': 'লৌরখ্রে',
      'milestone_need_score': '{0}% মথৌ তাই',
      'milestone_claim_reward': 'মানা লৌরৌ +{0} pts',
      'milestone_reward_toast': 'মানা লৌরখ্রে! +{0} pts',
      // FAQ Screen
      'help_faq': 'মচেৎ অমসুং FAQ',
      'common_questions': 'হংনবা ৱাহংশিং',
      'keep_blooming': 'মখা তানা সাতলু! 🌸',
      'faq_q1': 'ব্লুম (Bloom) হায়বসি করিনো?',
      'faq_a1':
          'ব্লুম হায়বসি \'গ্র্যাডেড এক্সপোজার\' কৌবা থৌওং অমগী খুত্থাংদা মীয়াম মরক্তা লৈবা অকিবশিং হন্থহননબા শেমখিবা ওপন সেলফ-হেল্প তুল অমনি। মচৌ খরা খক ওইবা মীয়াম মরক্কী থবকশিং লোইশিনবগী খুত্থাংদা, নহাক্কী মকোকনা মীয়াম মরক্তা লৈবা যোগাযোগশিং অসি অশোয়-অঙাম লৈতে অমসুং তৌবা ঙম্মি হায়না তাকই।',
      'faq_q2': 'লেভেলশিং অসি করম্না থবক তৌবগে?',
      'faq_a2':
          'ঐখোয়না \'মরূ\' (খ্বাইদগী লাইባ থবকশিং) দগী হৌগা \'সাতপা\' (অরুবা থবকশিং) ফাওবা চৎলি। নহাক্না থবক লোইশিনবগা থাজজগী পোਇন্ট ফংগনি। লেভেল ৱাংখৎলকপা মখেই পোਇন্ট হেন্না ফংগনি!',
      'faq_q3': 'থাজজগী মিতর হায়বসি করিনো?',
      'faq_a3':
          'নહા গী হোম স্ক্রিনদা লৈরিবা প্রোগ্রেস বার অসিনা নഹാ গী অপুনবা থাজবা উৎলি। নહাক্না থবক লোইশিনবগা মসি চাওখৎলকই। চেকশিনৌ: নહাক্না নুমিত খরনি প্র্যাকটিস তৌবা থক্লবদি থাজজগী স্কোর খরা হন্থরকপা য়াই, মসিনা থাজবা হায়বসি এক্সারসাইজ মথৌ তাবা মশা-মউ অমগুমনি হায়না পনখ্রে!',
      'faq_q4': 'মখা তাবা নুমিত (Streak) হায়বসি করিনো?',
      'faq_a4':
          'মসি নহাক্না সলগ ওইনা কমসে কম থবক অমা লোইশিনখ্রবা নুমিতশিংগী মশিংনি। অকিবা কোকহনবদা মখা তাবা অসিনা খ্বাইদগী মরুওইবা চাবিনি, মরম অদুনা নહા গী মঙাল অসি মুৎহনদনባ হোৎনৌ!',
      'faq_q5': 'ঐহাক্কী ডেটা কদাইদা থম্বগে?',
      'faq_a5':
          'নહા গী প্রাইভেসী অসি ঐখোয়গী অহানબા থবকনি। নહા গী খুമാঙ চাউশিনবা, পুৱারী অমসুং প্রোফাইল ডেটা পုံ পুম্নমক নਹਾ গী মশাগী ডিভাইস খকদা থম্মি। ক্লাউড সর্বরদা করিসু অপলোড তৌদে।',
      'faq_q6': 'অ্যাপ অসি ক্র্যাশ ওইরগদি করী তৌগদগে?',
      'faq_a6':
          'অ্যাপ অসি অরো ওইনা থবক তৌরগদি ফোন অসি রিস্টার্ট তৌও। অ্যাপ আপডেট তৌরবা মতুংদা এন্ড്രোইড সেটিংদগী অ্যাপ ক্যাশ (Cache) ক্লিয়র তৌবা মথৌ তারকপা য়াই। করিসু মায় পাকত্রবদি প্রোফাইলদগী \'খুമാঙ চাউশিনবা পুম্নमক રિસેટ તૌઓ\' শীজিন্নৌ।',
      'faq_q7': 'ঐহাক লেভেল স্কিপ তৌবা য়াব୍ରা?',
      'faq_a7':
          'য়াই! ঐখোয়না ধাপে ধাপে চৎনባ তাক্লবসু নհাক্কী হৌজিক লৈরিবা কনফোর্ট লেভেলগা চান্নবা ম্যাপদগী খুদিংਮক খল্লবা য়াই।',
      'faq_q8': 'থবক অসি খরা অরুবা ওইরগদি করী তৌগদগে?',
      'faq_a8':
          'ঐখোয়না থবক লোইশিননባ হোৎনবা তাক্লবসু অরুবা ওইরগদি হোম স্ক্রিনদা হল্লগা অমুক հন্না চঙলক্লগা হৌজিক লৈরিবা থবক অসি ওনবা য়াই।',
      'faq_q9': 'ঐখোয়ગા যোগাযোগ তৌবীয়ু',
      'faq_a9':
          'ঐখোয়গী অ্যাপ অসিগী মতাংদা অমসুং তুংগী আপডেটশিংগীদমক ইউজারশিংদগী সজেসন ফংবা পাম্মি। অ্যাপ অসিনা ইউজারশিংদা করম্না কান্নবগে, করী খক ৱাৎলিবগে অমসুং কদাইদা ফগৎহনবা মথৌ তאי হায়বসি খঙবা পাম্মি। চানবীদুনা feedback.bloom@gmail.com দা নહા গী মচেৎ পীবীয়ু, ঐখোয়না থমোইদগী হরাওগনি।',
      // Task Screen
      'stage_label': 'তাঙ্কক: {0}',
      'keep_growing': 'މখা তানা চাওখৎলু, {0}',
      'current_challenge': 'তাঙ্কক অসিগী হৌজিক লৈরিবা চ্যালেঞ্জ:',
      'stage_mastered': 'তাঙ্কক অসি মায় পাকখ্রে!',
      'all_done': 'নහাক্না তাঙ্কক অসিগী চ্যালেঞ্জ ਪুম্নমক লোইশিনখ্রে।',
      'return_map': 'ম্যাপতা হল্লু',
      'well_done': 'য়াম্না ফৈ!',
      'i_completed': 'ঐহাক মসি लोइशिनখ্রে',
      'level_up_suggestion_title': 'লেভেল হেনগৎনબા সজেসন',
      'level_up_suggestion_message':
          'নহাক্না লেভেল অসিদা থবক ১০ লোইশিনখ্রে! নհাক তুংগী লেভেলগীদমক শেম-শারে। মখা তানা ৱাংখৎপা পাম্ব্রা?',
      'stay_here': 'মফম অসিদা লৈয়ু',
      'move_to_next_level': 'তুংগী লেভেলদা চৎলু',
      // Reflection Screen
      'reflect': 'নਹਾ গী চাওখৎপগী মতাংদা খন্নৌ',
      'challenge': 'ꯆꯤꯡꯅꯕ',
      'anxiety_q': 'নஹাক করম্না অকিবা ফাওখিবগে? (১-১০)',
      'what_happened': 'অশেংবদা করী থোকখිබগে?',
      'write_experience_hint': 'ਨਹਾ গী এক্সপিরিয়েন্স অসি ইম্মু...',
      'finish': 'ખন্নবা লোইশিনখ্রে',
      // General / Auth
      'welcome': 'ব্লুমদা ওৱেলকম তৌবীয়ু',
      'subtitle': 'নਹਾ গী থাজবা হেনগৎহন্নবা অশোয়-অঙাম লৈতবা মফम অমা।',
      'start': 'ঐহাক্কী খোঙচৎ হৌও',
      'guest': 'গেস্ট ওইনা চৎলু',
      'hello': 'খুরুমজরি',
      'profile': 'ঐহাক্কী প্রোফাইল',
      'history': 'ঐহাক্কী চাওখৎপগী খোঙচৎ',
      'streak': 'হৌজিক লৈরিবা মখা তাবা নุมিত',
      'best': 'খ্বাইদগী ফবা মখা তাবা নুমিত',
      'points': 'ਥাজজগী পোਇন্টশিং',
      //Splash Screen
      'loading': 'নહા গী বাগান লোড তৌরি...',
    },
    'my': {
      // Progress Screen
      'your_journey': 'Perjalanan Anda',
      'keep_growing_sub':
          'Setiap langkah kecil adalah kemenangan. Teruskan berkembang!',
      'how_it_works': 'Bagaimana ia berfungsi?',
      'total_points': 'Jumlah Mata',
      'current_streak': 'Rentetan Semasa',
      'tasks_done': 'Tugasan Selesai',
      'rank': 'Pangkat',
      'days': 'Hari',
      'contact_us': 'Hubungi Kami',
      'contact_email_prompt': 'Untuk sokongan dan maklum balas, e-mel kami di:',
      'close': 'Tutup',
      // Level Map Screen
      'tap_to_view_journey': 'Ketik untuk melihat perjalanan anda! 🌸',
      'tap_to_start': 'Ketik untuk memulakan cabaran',
      'choose_level': 'Pilih peringkat perkembangan anda:',
      'view_journey': 'Lihat Perjalanan Perkembangan',
      'progress': 'Kemajuan Perkembangan Anda',
      'level_seedling': 'Anak Benih (Seedling)',
      'level_sprout': 'Tunas (Sprout)',
      'level_leaf': 'Daun (Leaf)',
      'level_stem': 'Batang (Stem)',
      'level_bloom': 'Mekar (Bloom)',
      // Profile Screen
      'account': 'Akaun',
      'display_name': 'Nama Paparan',
      'save_name': 'Simpan Nama',
      'app_theme': 'Tema Aplikasi',
      'select_color': 'Pilih Warna Bloom Anda:',
      'light_mode': 'Cerah',
      'dark_mode': 'Gelap',
      'language': 'Bahasa',
      'logout': 'Log Keluar',
      'profile_updated': 'Profil dikemas kini!',
      'pick_theme_color': 'Pilih warna tema',
      'done': 'Selesai',
      'reset_all_progress': 'Set Semula Semua Kemajuan',
      'reset_confirm_title': 'Adakah anda pasti?',
      'reset_confirm_message':
          'Ini akan memadamkan skor keyakinan, rentetan, dan semua sejarah anda secara kekal. Tindakan ini tidak boleh ditarik balik.',
      'cancel': 'Batal',
      'reset_everything': 'Set Semula Segalanya',
      'reset_success': 'Semua kemajuan telah diset semula.',
      // Shop Screen
      'shop_title': 'Kedai Bloom',
      'your_points': 'Mata Anda',
      'available_items': 'Item yang Tersedia',
      'streak_freeze': 'Pembeku Rentetan',
      'protects_streak': 'Melindungi rentetan daripada diset semula',
      'your_inventory': 'Inventori Anda',
      'owned': 'Dimiliki',
      'equipped': 'Dilengkapi',
      'equip_freeze': 'Lengkapi Pembeku',
      'buy': 'Beli',
      'freeze_purchased': 'Pembeku Berjaya Dibeli!',
      'not_enough_points': 'Mata tidak mencukupi!',
      'freeze_equipped': 'Pembeku dilengkapi!',
      // Milestone Screen
      'your_growth_path': 'Laluan Perkembangan Anda',
      'the_awakening': 'Kesedaran',
      'seed_badge': 'Lencana Benih',
      'first_spark': 'Percikan Pertama',
      'bronze_leaf': 'Daun Gangsa',
      'social_courage': 'Keberanian Sosial',
      'silver_sprout': 'Tunas Perak',
      'confidence_bloom': 'Keyakinan Mekar',
      'gold_flower': 'Bunga Emas',
      'mastery': 'Penguasaan',
      'diamond_crown': 'Mahkota Berlian',
      'milestone_claimed': 'Ditebus',
      'milestone_need_score': 'Memerlukan {0}%',
      'milestone_claim_reward': 'Tebus +{0} mata',
      'milestone_reward_toast': 'Berjaya ditebus! +{0} mata',
      // FAQ Screen
      'help_faq': 'Bantuan dan Soalan Lazim',
      'common_questions': 'Soalan Lazim',
      'keep_blooming': 'Teruskan Mekar! 🌸',
      'faq_q1': 'Apakah itu Bloom?',
      'faq_a1':
          'Bloom ialah alat bantu diri yang direka untuk membantu orang ramai mengurangkan kegelisahan sosial melalui proses yang dipanggil \'Pendedahan Berperingkat.\' Dengan menyelesaikan tugasan sosial yang kecil dan boleh diurus, anda melatih otak anda untuk menyedari bahawa interaksi sosial adalah selamat dan boleh dikendalikan.',
      'faq_q2': 'Bagaimanakah tahap-tahap ini berfungsi?',
      'faq_a2':
          'Kita bermula dengan \'Anak Benih\' (tugasan yang sangat mudah) dan meningkat ke \'Mekar\' (tugasan yang lebih mencabar). Apabila anda menyelesaikan tugasan, anda akan mendapat mata keyakinan. Semakin tinggi tahap, semakin banyak mata yang anda perolehi!',
      'faq_q3': 'Apakah itu Meter Keyakinan?',
      'faq_a3':
          'Bar kemajuan pada skrin utama anda mewakili keseluruhan keyakinan anda. Ia berkembang apabila anda menyelesaikan tugasan. Berhati-hati: jika anda berhenti berlatih selama beberapa hari, skor keyakinan anda mungkin merosot sedikit, mengingatkan anda bahawa keyakinan ialah otot yang memerlukan latihan yang kerap!',
      'faq_q4': 'Apakah itu Rentetan (Streak)?',
      'faq_a4':
          'Rentetan ialah bilangan hari berturut-turut anda telah menyelesaikan sekurang-kurangnya satu tugasan. Konsistensi adalah kunci untuk mengatasi kegelisahan, jadi cuba pastikan api anda terus menyala!',
      'faq_q5': 'Di manakah data saya disimpan?',
      'faq_a5':
          'Privasi anda adalah keutamaan kami. Semua kemajuan, sejarah, dan data profil anda disimpan secara tempatan pada peranti anda sendiri. Tiada apa-apa yang dimuat naik ke pelayan awan.',
      'faq_q6':
          'Apakah yang perlu saya lakukan jika aplikasi terhenti (crash)?',
      'faq_a6':
          'Jika aplikasi berkelakuan pelik, cuba mulakan semula telefon anda. Jika anda telah mengemas kini aplikasi, anda mungkin perlu mengosongkan cache aplikasi dalam tetapan Android anda. Jika semuanya gagal, anda boleh menggunakan pilihan \'Set Semula Semua Kemajuan\' dalam Profil anda.',
      'faq_q7': 'Bolehkah saya melangkau tahap?',
      'faq_a7':
          'Ya! Walaupun kami mengesyorkan laluan berperingkat, anda bebas memilih mana-mana tahap daripada peta yang dirasakan sesuai dengan tahap keselesaan semasa anda.',
      'faq_q8': 'Bagaimana jika tugasan itu sangat sukar?',
      'faq_a8':
          'Walaupun kami mengesyorkan anda untuk cuba menyelesaikan tugasan tersebut, anda boleh kembali ke skrin utama, dan masuk semula untuk menukar tugasan semasa.',
      'faq_q9': 'Hubungi Kami',
      'faq_a9':
          'Kami sangat suka mendengar pendapat daripada pengguna tentang aplikasi kami, serta cadangan untuk kemas kini masa hadapan. Kami ingin tahu sejauh mana aplikasi ini membantu pengguna, apa yang kurang dan apa yang memerlukan penambahbaikan. Sila kongsi maklum balas anda di feedback.bloom@gmail.com, kami amat menghargainya.',
      // Task Screen
      'stage_label': 'Peringkat: {0}',
      'keep_growing': 'Teruskan berkembang, {0}',
      'current_challenge': 'Cabaran semasa anda untuk peringkat ini:',
      'stage_mastered': 'Peringkat Berjaya Dikuasai!',
      'all_done': 'Anda telah menyelesaikan semua cabaran dalam peringkat ini.',
      'return_map': 'Kembali ke Peta',
      'well_done': 'Syabas!',
      'i_completed': 'Saya Telah Menyelesaikan Ini',
      'level_up_suggestion_title': 'Cadangan Naik Tahap',
      'level_up_suggestion_message':
          'Anda telah menyelesaikan 10 tugasan di tahap ini! Anda sudah bersedia untuk tahap seterusnya. Mahu naik tahap?',
      'stay_here': 'Kekal Di Sini',
      'move_to_next_level': 'Pergi ke Tahap Seterusnya',
      // Reflection Screen
      'reflect': 'Refleksi tentang perkembangan anda',
      'challenge': 'Cabaran',
      'anxiety_q': 'Sejauh mana anda berasa gelisah? (1-10)',
      'what_happened': 'Apa yang sebenarnya berlaku?',
      'write_experience_hint': 'Tulis tentang pengalaman anda...',
      'finish': 'Selesai Refleksi',
      // General / Auth
      'welcome': 'Selamat Datang ke Bloom',
      'subtitle': 'Ruang selamat untuk mengembangkan keyakinan anda.',
      'start': 'Mula Perjalanan Saya',
      'guest': 'Teruskan sebagai Tetamu',
      'hello': 'Hello',
      'profile': 'Profil Saya',
      'history': 'Perjalanan Perkembangan Saya',
      'streak': 'Rentetan Semasa',
      'best': 'Rentetan Terbaik',
      'points': 'Mata Keyakinan',
      //Splash Screen
      'loading': 'Memuatkan taman anda...',
    },
    'th': {
      // Progress Screen
      'your_journey': 'เส้นทางการเติบโตของคุณ',
      'keep_growing_sub': 'ทุกก้าวเล็กๆ คือชัยชนะ เติบโตต่อไปนะ!',
      'how_it_works': 'มันทำงานอย่างไร?',
      'total_points': 'คะแนนรวม',
      'current_streak': 'ความต่อเนื่องปัจจุบัน',
      'tasks_done': 'ภารกิจที่สำเร็จ',
      'rank': 'ระดับ (Rank)',
      'days': 'วัน',
      'contact_us': 'ติดต่อเรา',
      'contact_email_prompt':
          'สำหรับความช่วยเหลือและข้อเสนอแนะ โปรดอีเมลหาเราที่:',
      'close': 'ปิด',
      // Level Map Screen
      'tap_to_view_journey': 'แตะเพื่อดูเส้นทางการเติบโตของคุณ! 🌸',
      'tap_to_start': 'แตะเพื่อเริ่มความท้าทาย',
      'choose_level': 'เลือกขั้นการเติบโตของคุณ:',
      'view_journey': 'ดูเส้นทางการเติบโต',
      'progress': 'ความคืบหน้าการเติบโตของคุณ',
      'level_seedling': 'ต้นกล้า (Seedling)',
      'level_sprout': 'ต้นอ่อน (Sprout)',
      'level_leaf': 'ใบไม้ (Leaf)',
      'level_stem': 'ลำต้น (Stem)',
      'level_bloom': 'เบ่งบาน (Bloom)',
      // Profile Screen
      'account': 'บัญชี',
      'display_name': 'ชื่อที่แสดง',
      'save_name': 'บันทึกชื่อ',
      'app_theme': 'ธีมแอป',
      'select_color': 'เลือกสีประจำตัวของคุณ:',
      'light_mode': 'โหมดสว่าง',
      'dark_mode': 'โหมดมืด',
      'language': 'ภาษา',
      'logout': 'ออกจากระบบ',
      'profile_updated': 'อัปเดตโปรไฟล์เรียบร้อยแล้ว!',
      'pick_theme_color': 'เลือกสีธีม',
      'done': 'เสร็จสิ้น',
      'reset_all_progress': 'รีเซ็ตความคืบหน้าทั้งหมด',
      'reset_confirm_title': 'คุณแน่ใจหรือไม่?',
      'reset_confirm_message':
          'การดำเนินการนี้จะลบคะแนนความมั่นใจ คะแนนความต่อเนื่อง และประวัติทั้งหมดของคุณอย่างถาวร โดยไม่สามารถเรียกคืนได้',
      'cancel': 'ยกเลิก',
      'reset_everything': 'รีเซ็ตทุกอย่าง',
      'reset_success': 'รีเซ็ตความคืบหน้าทั้งหมดแล้ว',
      // Shop Screen
      'shop_title': 'ร้านค้า Bloom',
      'your_points': 'คะแนนของคุณ',
      'available_items': 'ไอเทมที่มีให้เลือก',
      'streak_freeze': 'การ์ดแช่แข็งความต่อเนื่อง',
      'protects_streak': 'ป้องกันไม่ให้คะแนนความต่อเนื่องถูกรีเซ็ต',
      'your_inventory': 'กระเป๋าเก็บของของคุณ',
      'owned': 'ครอบครองแล้ว',
      'equipped': 'ติดตั้งอยู่',
      'equip_freeze': 'ติดตั้งการ์ดแช่แข็ง',
      'buy': 'ซื้อ',
      'freeze_purchased': 'ซื้อการ์ดแช่แข็งสำเร็จแล้ว!',
      'not_enough_points': 'คะแนนไม่เพียงพอ!',
      'freeze_equipped': 'ติดตั้งการ์ดแช่แข็งแล้ว!',
      // Milestone Screen
      'your_growth_path': 'เส้นทางสู่ความสำเร็จของคุณ',
      'the_awakening': 'การตื่นรู้',
      'seed_badge': 'ตราสัญลักษณ์เมล็ดพันธุ์',
      'first_spark': 'ประกายไฟแรก',
      'bronze_leaf': 'ใบไม้ทองแดง',
      'social_courage': 'ความกล้าหาญในสังคม',
      'silver_sprout': 'ต้นอ่อนเงิน',
      'confidence_bloom': 'ความมั่นใจผลิบาน',
      'gold_flower': 'ดอกไม้ทองคำ',
      'mastery': 'ปรมาจารย์',
      'diamond_crown': 'มงกุฎเพชร',
      'milestone_claimed': 'รับแล้ว',
      'milestone_need_score': 'ต้องการ {0}%',
      'milestone_claim_reward': 'รับรางวัล +{0} คะแนน',
      'milestone_reward_toast': 'รับรางวัลสำเร็จแล้ว! +{0} คะแนน',
      // FAQ Screen
      'help_faq': 'ความช่วยเหลือ & FAQ',
      'common_questions': 'คำถามที่พบบ่อย',
      'keep_blooming': 'เบ่งบานต่อไปนะ! 🌸',
      'faq_q1': 'Bloom คืออะไร?',
      'faq_a1':
          'Bloom คือเครื่องมือช่วยเหลือตนเองที่ออกแบบมาเพื่อช่วยลดความวิตกกังวลทางสังคมผ่านกระบวนการที่เรียกว่า \'Graded Exposure\' (การค่อยๆ เผชิญหน้ากับสิ่งที่กลัว) โดยการทำภารกิจทางสังคมเล็กๆ ที่จัดการได้ง่ายให้สำเร็จ คุณจะช่วยฝึกสมองให้รับรู้ว่าการปฏิสัมพันธ์ในสังคมนั้นปลอดภัยและสามารถรับมือได้',
      'faq_q2': 'แต่ละระดับทำงานอย่างไร?',
      'faq_a2':
          'เราเริ่มต้นจากระดับ \'ต้นกล้า\' (ภารกิจที่ง่ายมาก) และขยับขึ้นไปจนถึงระดับ \'เบ่งบาน\' (ภารกิจที่มีความท้าทายมากขึ้น) เมื่อคุณทำภารกิจสำเร็จ คุณจะได้รับคะแนนความมั่นใจ ยิ่งระดับสูงขึ้น คะแนนที่คุณจะได้รับก็ยิ่งมากขึ้น!',
      'faq_q3': 'Confidence Meter (มิเตอร์ความมั่นใจ) คืออะไร?',
      'faq_a3':
          'แถบความคืบหน้าบนหน้าจอหลักแสดงถึงความมั่นใจโดยรวมของคุณ ซึ่งจะเติบโตขึ้นเมื่อคุณทำภารกิจสำเร็จ แต่โปรดระวัง: หากคุณหยุดฝึกฝนเป็นเวลาหลายวัน คะแนนความมั่นใจของคุณอาจลดลงเล็กน้อย เพื่อเตือนให้คุณรู้ว่าความมั่นใจก็เหมือนกล้ามเนื้อที่ต้องออกกำลังกายอย่างสม่ำเสมอ!',
      'faq_q4': 'Streak (ความต่อเนื่อง) คืออะไร?',
      'faq_a4':
          'Streak คือการนับจำนวนวันติดต่อกันที่คุณทำภารกิจสำเร็จอย่างน้อยหนึ่งภารกิจ ความสม่ำเสมอคือหัวใจสำคัญในการเอาชนะความวิตกกังวล ดังนั้นพยายามรักษาเปลวไฟของคุณให้โชติช่วงอยู่เสมอนะ!',
      'faq_q5': 'ข้อมูลของฉันถูกเก็บไว้ที่ไหน?',
      'faq_a5':
          'ความเป็นส่วนตัวของคุณคือสิ่งที่เราให้ความสำคัญที่สุด ความคืบหน้า ประวัติ และข้อมูลโปรไฟล์ทั้งหมดของคุณจะถูกจัดเก็บไว้ในอุปกรณ์ของคุณเองเท่านั้น โดยไม่มีการอัปโหลดไปยังเซิร์ฟเวอร์คลาวด์ใดๆ',
      'faq_q6': 'ต้องทำอย่างไรหากแอปคราช (แอปเด้ง)?',
      'faq_a6':
          'หากแอปทำงานผิดปกติ ลองรีสตาร์ทโทรศัพท์ของคุณ หากคุณเพิ่งอัปเดตแอป คุณอาจต้องล้างแคชของแอปในการตั้งค่าระบบ Android ของคุณ หากลองทุกวิธีแล้วยังไม่ได้ผล คุณสามารถใช้ตัวเลือก \'รีเซ็ตความคืบหน้าทั้งหมด\' ในหน้าโปรไฟล์ของคุณได้',
      'faq_q7': 'ฉันสามารถข้ามระดับได้หรือไม่?',
      'faq_a7':
          'ได้แน่นอน! แม้ว่าเราจะแนะนำให้เดินไปตามเส้นทางทีละขั้น แต่คุณก็มีอิสระที่จะเลือกปรับระดับใดก็ได้จากแผนที่ที่รู้สึกว่าเหมาะสมกับความสบายใจของคุณในปัจจุบัน',
      'faq_q8': 'หากภารกิจนั้นยากเกินไปล่ะ?',
      'faq_a8':
          'แม้ว่าเราจะแนะนำให้คุณพยายามทำภารกิจให้สำเร็จ แต่ถ้ามันยากเกินไปจริงๆ คุณสามารถกลับไปที่หน้าจอหลักแล้วกดเข้ามาใหม่เพื่อเปลี่ยนภารกิจปัจจุบันได้',
      'faq_q9': 'ติดต่อเรา',
      'faq_a9':
          'พวกเรายินดีที่จะรับฟังความคิดเห็นจากผู้ใช้เกี่ยวกับแอปของเรา รวมถึงข้อเสนอแนะสำหรับการอัปเดตในอนาคต พวกเราอยากรู้ว่าแอปนี้ช่วยผู้ใช้ได้ดีแค่ไหน มีส่วนไหนที่ยังขาดหายหรือต้องการการปรับปรุง โปรดแชร์ข้อเสนอแนะของคุณมาที่ feedback.bloom@gmail.com พวกเราจะขอบคุณเป็นอย่างยิ่ง',
      // Task Screen
      'stage_label': 'ขั้น: {0}',
      'keep_growing': 'เติบโตต่อไปนะ คุณ {0}',
      'current_challenge': 'ความท้าทายปัจจุบันของคุณสำหรับขั้นนี้:',
      'stage_mastered': 'ผ่านขั้นนี้สำเร็จแล้ว!',
      'all_done': 'คุณทำความท้าทายทั้งหมดในขั้นนี้สำเร็จแล้ว',
      'return_map': 'กลับไปที่แผนที่',
      'well_done': 'ยอดเยี่ยมมาก!',
      'i_completed': 'ฉันทำภารกิจนี้สำเร็จแล้ว',
      'level_up_suggestion_title': 'คำแนะนำในการเลื่อนระดับ',
      'level_up_suggestion_message':
          'คุณทำภารกิจในระดับนี้สำเร็จครบ 10 ภารกิจแล้ว! คุณพร้อมสำหรับระดับต่อไปแล้วล่ะ ต้องการเลื่อนระดับเลยไหม?',
      'stay_here': 'อยู่ที่นี่ต่อก่อน',
      'move_to_next_level': 'เลื่อนไปยังระดับถัดไป',
      // Reflection Screen
      'reflect': 'ทบทวนการเติบโตของคุณ',
      'challenge': 'ท้าทาย',
      'anxiety_q': 'คุณรู้สึกวิตกกังวลแค่ไหน? (1-10)',
      'what_happened': 'เกิดอะไรขึ้นจริงบ้าง?',
      'write_experience_hint': 'เขียนบอกเล่าเกี่ยวกับประสบการณ์ของคุณ...',
      'finish': 'เสร็จสิ้นการทบทวน',
      // General / Auth
      'welcome': 'ยินดีต้อนรับสู่ Bloom',
      'subtitle': 'พื้นที่ปลอดภัยในการบ่มเพาะความมั่นใจของคุณ',
      'start': 'เริ่มเส้นทางของฉัน',
      'guest': 'ดำเนินการต่อในฐานะผู้เยี่ยมชม',
      'hello': 'สวัสดี',
      'profile': 'โปรไฟล์ของฉัน',
      'history': 'บันทึกการเติบโตของฉัน',
      'streak': 'ความต่อเนื่องปัจจุบัน',
      'best': 'ความต่อเนื่องที่ดีที่สุด',
      'points': 'คะแนนความมั่นใจ',
      //Splash Screen
      'loading': 'กำลังโหลดสวนของคุณ...',
    },
    'vi': {
      // Progress Screen
      'your_journey': 'Hành trình của bạn',
      'keep_growing_sub':
          'Mỗi bước đi nhỏ đều là một chiến thắng. Hãy tiếp tục phát triển!',
      'how_it_works': 'Ứng dụng hoạt động như thế nào?',
      'total_points': 'Tổng điểm',
      'current_streak': 'Chuỗi ngày hiện tại',
      'tasks_done': 'Thử thách đã hoàn thành',
      'rank': 'Hạng',
      'days': 'Ngày',
      'contact_us': 'Liên hệ với chúng tôi',
      'contact_email_prompt':
          'Để được hỗ trợ và góp ý, hãy gửi email cho chúng tôi tại:',
      'close': 'Đóng',
      // Level Map Screen
      'tap_to_view_journey': 'Nhấn để xem hành trình của bạn! 🌸',
      'tap_to_start': 'Nhấn để bắt đầu thử thách',
      'choose_level': 'Chọn giai đoạn phát triển của bạn:',
      'view_journey': 'Xem hành trình phát triển',
      'progress': 'Tiến trình phát triển của bạn',
      'level_seedling': 'Gieo hạt (Seedling)',
      'level_sprout': 'Nảy mầm (Sprout)',
      'level_leaf': 'Ra lá (Leaf)',
      'level_stem': 'Vươn cành (Stem)',
      'level_bloom': 'Nở hoa (Bloom)',
      // Profile Screen
      'account': 'Tài khoản',
      'display_name': 'Tên hiển thị',
      'save_name': 'Lưu tên',
      'app_theme': 'Giao diện ứng dụng',
      'select_color': 'Chọn màu sắc Bloom của bạn:',
      'light_mode': 'Sáng',
      'dark_mode': 'Tối',
      'language': 'Ngôn ngữ',
      'logout': 'Đăng xuất',
      'profile_updated': 'Hồ sơ đã được cập nhật!',
      'pick_theme_color': 'Chọn một màu chủ đề',
      'done': 'Xong',
      'reset_all_progress': 'Xóa toàn bộ tiến trình',
      'reset_confirm_title': 'Bạn có chắc chắn không?',
      'reset_confirm_message':
          'Hành động này sẽ xóa vĩnh viễn điểm tự tin, chuỗi ngày và tất cả lịch sử của bạn. Không thể hoàn tác.',
      'cancel': 'Hủy',
      'reset_everything': 'Xóa tất cả',
      'reset_success': 'Toàn bộ tiến trình đã được đặt lại.',
      // Shop Screen
      'shop_title': 'Cửa hàng Bloom',
      'your_points': 'Điểm của bạn',
      'available_items': 'Vật phẩm có sẵn',
      'streak_freeze': 'Băng bảo vệ chuỗi',
      'protects_streak': 'Bảo vệ chuỗi ngày không bị đặt lại về 0',
      'your_inventory': 'Túi đồ của bạn',
      'owned': 'Đã sở hữu',
      'equipped': 'Đang trang bị',
      'equip_freeze': 'Trang bị Băng bảo vệ',
      'buy': 'Mua',
      'freeze_purchased': 'Đã mua Băng bảo vệ chuỗi!',
      'not_enough_points': 'Không đủ điểm!',
      'freeze_equipped': 'Đã trang bị Băng bảo vệ!',
      // Milestone Screen
      'your_growth_path': 'Con đường phát triển của bạn',
      'the_awakening': 'Thức tỉnh',
      'seed_badge': 'Huy hiệu Hạt giống',
      'first_spark': 'Tia sáng đầu tiên',
      'bronze_leaf': 'Lá đồng',
      'social_courage': 'Dũng khí xã hội',
      'silver_sprout': 'Mầm bạc',
      'confidence_bloom': 'Tự tin tỏa sáng',
      'gold_flower': 'Hoa vàng',
      'mastery': 'Làm chủ',
      'diamond_crown': 'Vương miện kim cương',
      'milestone_claimed': 'Đã nhận',
      'milestone_need_score': 'Cần {0}%',
      'milestone_claim_reward': 'Nhận +{0} điểm',
      'milestone_reward_toast': 'Đã nhận! +{0} điểm',
      // FAQ Screen
      'help_faq': 'Trợ giúp & Câu hỏi thường gặp',
      'common_questions': 'Câu hỏi thường gặp',
      'keep_blooming': 'Hãy luôn tỏa sáng! 🌸',
      'faq_q1': 'Bloom là gì?',
      'faq_a1':
          'Bloom là một công cụ tự trợ giúp được thiết kế để giúp mọi người giảm bớt sự lo âu xã hội thông qua một quá trình gọi là \'Tiếp xúc dần dần\' (Graded Exposure). Bằng cách hoàn thành các nhiệm vụ xã hội nhỏ, vừa sức, bạn sẽ huấn luyện bộ não nhận ra rằng các tương tác xã hội là an toàn và có thể kiểm soát được.',
      'faq_q2': 'Các cấp độ hoạt động như thế nào?',
      'faq_a2':
          'Chúng ta bắt đầu với \'Gieo hạt\' (các nhiệm vụ rất dễ) và nâng dần lên \'Nở hoa\' (các nhiệm vụ thử thách hơn). Khi hoàn thành nhiệm vụ, bạn sẽ tích lũy được điểm tự tin. Cấp độ càng cao, bạn càng nhận được nhiều điểm!',
      'faq_q3': 'Thước đo tự tin (Confidence Meter) là gì?',
      'faq_a3':
          'Thanh tiến trình trên màn hình chính đại diện cho mức độ tự tin tổng thể của bạn. Nó sẽ tăng lên khi bạn hoàn thành các thử thách. Hãy cẩn thận: nếu bạn ngừng luyện tập trong vài ngày, điểm tự tin của bạn có thể bị giảm nhẹ, như một lời nhắc nhở rằng tự tin là một khối cơ bắp cần được rèn luyện thường xuyên!',
      'faq_q4': 'Chuỗi ngày (Streak) là gì?',
      'faq_a4':
          'Chuỗi ngày là số ngày liên tiếp bạn hoàn thành ít nhất một thử thách. Sự kiên trì là chìa khóa để vượt qua lo âu, vì vậy hãy cố gắng giữ cho ngọn lửa của bạn luôn cháy nhé!',
      'faq_q5': 'Dữ liệu của tôi được lưu trữ ở đâu?',
      'faq_a5':
          'Quyền riêng tư của bạn là ưu tiên hàng đầu của chúng tôi. Tất cả tiến trình, lịch sử và dữ liệu hồ sơ của bạn đều được lưu trữ cục bộ trên chính thiết bị của bạn. Không có dữ liệu nào được tải lên máy chủ đám mây.',
      'faq_q6': 'Tôi phải làm gì nếu ứng dụng bị lỗi (crash)?',
      'faq_a6':
          'Nếu ứng dụng hoạt động bất thường, hãy thử khởi động lại điện thoại của bạn. Nếu bạn vừa cập nhật ứng dụng, bạn có thể cần xóa bộ nhớ đệm (cache) của ứng dụng trong cài đặt Android. Nếu mọi cách đều thất bại, bạn có chơi tùy chọn \'Xóa toàn bộ tiến trình\' trong trang Cá nhân của mình.',
      'faq_q7': 'Tôi có thể bỏ qua các cấp độ không?',
      'faq_a7':
          'Có chứ! Mặc dù chúng tôi khuyên bạn nên đi theo lộ trình từng bước, bạn hoàn toàn có thể tự do chọn bất kỳ cấp độ nào trên bản đồ phù hợp với mức độ thoải mái hiện tại của mình.',
      'faq_q8': 'Nếu thử thách quá khó thì sao?',
      'faq_a8':
          'Mặc dù chúng tôi khuyên bạn nên cố gắng hoàn thành thử thách, nhưng nếu nó thực sự quá khó, bạn chỉ cần quay lại màn hình chính và nhấn vào lại để đổi sang một thử thách khác.',
      'faq_q9': 'Liên hệ với chúng tôi',
      'faq_a9':
          'Chúng tôi rất mong nhận được những chia sẻ của người dùng về ứng dụng, cũng như các đề xuất cho những bản cập nhật trong tương lai. Chúng tôi muốn biết ứng dụng hỗ trợ bạn tốt đến mức nào, còn thiếu sót gì và cần cải thiện ở đâu. Xin vui lòng gửi phản hồi về địa chỉ feedback.bloom@gmail.com, chúng tôi vô cùng trân trọng.',
      // Task Screen
      'stage_label': 'Giai đoạn: {0}',
      'keep_growing': 'Hãy tiếp tục phát triển nhé, {0}',
      'current_challenge': 'Thử thách hiện tại của bạn ở giai đoạn này:',
      'stage_mastered': 'Đã làm chủ giai đoạn này!',
      'all_done': 'Bạn đã hoàn thành tất cả các thử thách trong giai đoạn này.',
      'return_map': 'Quay lại Bản đồ',
      'well_done': 'Làm tốt lắm!',
      'i_completed': 'Tôi đã hoàn thành thử thách này',
      'level_up_suggestion_title': 'Gợi ý lên cấp',
      'level_up_suggestion_message':
          'Bạn đã hoàn thành 10 thử thách ở cấp độ này! Bạn đã sẵn sàng cho cấp độ tiếp theo. Bạn có muốn tiến lên không?',
      'stay_here': 'Ở lại đây thêm chút',
      'move_to_next_level': 'Tiến lên cấp độ tiếp theo',
      // Reflection Screen
      'reflect': 'Suy ngẫm về sự phát triển của bạn',
      'challenge': 'Thử thách',
      'anxiety_q': 'Bạn đã cảm thấy lo lắng ở mức nào? (1-10)',
      'what_happened': 'Điều gì đã thực sự xảy ra?',
      'write_experience_hint': 'Viết về trải nghiệm của bạn...',
      'finish': 'Hoàn thành suy ngẫm',
      // General / Auth
      'welcome': 'Chào mừng đến với Bloom',
      'subtitle': 'Một không gian an toàn để nuôi dưỡng lòng tự tin của bạn.',
      'start': 'Bắt đầu hành trình',
      'guest': 'Tiếp tục với tư cách Khách',
      'hello': 'Xin chào',
      'profile': 'Hồ sơ của tôi',
      'history': 'Hành trình phát triển',
      'streak': 'Chuỗi ngày hiện tại',
      'best': 'Chuỗi ngày tốt nhất',
      'points': 'Điểm tự tin',
      //Splash Screen
      'loading': 'Đang tải khu vườn của bạn...',
    },
    'id': {
      // Progress Screen
      'your_journey': 'Perjalananmu',
      'keep_growing_sub':
          'Setiap langkah kecil adalah kemenangan. Teruslah berkembang!',
      'how_it_works': 'Bagaimana cara kerjanya?',
      'total_points': 'Total Poin',
      'current_streak': 'Rentetan Hari Ini',
      'tasks_done': 'Tantangan Selesai',
      'rank': 'Peringkat',
      'days': 'Hari',
      'contact_us': 'Hubungi Kami',
      'contact_email_prompt':
          'Untuk dukungan dan saran, kirim email kepada kami di:',
      'close': 'Tutup',
      // Level Map Screen
      'tap_to_view_journey': 'Ketuk untuk melihat perjalananmu! 🌸',
      'tap_to_start': 'Ketuk untuk memulai tantangan',
      'choose_level': 'Pilih tahap perkembanganmu:',
      'view_journey': 'Lihat Perjalanan Perkembangan',
      'progress': 'Kemajuan Perkembanganmu',
      'level_seedling': 'Bibit (Seedling)',
      'level_sprout': 'Tunas (Sprout)',
      'level_leaf': 'Daun (Leaf)',
      'level_stem': 'Batang (Stem)',
      'level_bloom': 'Mekar (Bloom)',
      // Profile Screen
      'account': 'Akun',
      'display_name': 'Nama Tampilan',
      'save_name': 'Simpan Nama',
      'app_theme': 'Tema Aplikasi',
      'select_color': 'Pilih Warna Bloom-mu:',
      'light_mode': 'Terang',
      'dark_mode': 'Gelap',
      'language': 'Bahasa',
      'logout': 'Keluar',
      'profile_updated': 'Profil berhasil diperbarui!',
      'pick_theme_color': 'Pilih warna tema',
      'done': 'Selesai',
      'reset_all_progress': 'Hapus Semua Kemajuan',
      'reset_confirm_title': 'Apakah kamu yakin?',
      'reset_confirm_message':
          'Tindakan ini akan menghapus skor kepercayaan diri, rentetan hari, dan semua riwayatmu secara permanen. Tindakan ini tidak dapat dibatalkan.',
      'cancel': 'Batal',
      'reset_everything': 'Hapus Semua',
      'reset_success': 'Semua kemajuan telah diset ulang.',
      // Shop Screen
      'shop_title': 'Toko Bloom',
      'your_points': 'Poinmu',
      'available_items': 'Item yang Tersedia',
      'streak_freeze': 'Pembeku Rentetan',
      'protects_streak': 'Melindungi rentetan hari agar tidak kembali ke nol',
      'your_inventory': 'Inventarismu',
      'owned': 'Dimiliki',
      'equipped': 'Terpasang',
      'equip_freeze': 'Pasang Pembeku',
      'buy': 'Beli',
      'freeze_purchased': 'Pembeku Rentetan Berhasil Dibeli!',
      'not_enough_points': 'Poin tidak mencukupi!',
      'freeze_equipped': 'Pembeku berhasil dipasang!',
      // Milestone Screen
      'your_growth_path': 'Jalur Perkembanganmu',
      'the_awakening': 'Kesadaran',
      'seed_badge': 'Lencana Benih',
      'first_spark': 'Percikan Pertama',
      'bronze_leaf': 'Daun Perunggu',
      'social_courage': 'Keberanian Sosial',
      'silver_sprout': 'Tunas Perak',
      'confidence_bloom': 'Keyakinan Mekar',
      'gold_flower': 'Bunga Emas',
      'mastery': 'Penguasaan',
      'diamond_crown': 'Mahkota Berlian',
      'milestone_claimed': 'Diambil',
      'milestone_need_score': 'Butuh {0}%',
      'milestone_claim_reward': 'Ambil +{0} poin',
      'milestone_reward_toast': 'Berhasil diambil! +{0} poin',
      // FAQ Screen
      'help_faq': 'Bantuan & FAQ',
      'common_questions': 'Pertanyaan Umum',
      'keep_blooming': 'Teruslah Mekar! 🌸',
      'faq_q1': 'Apa itu Bloom?',
      'faq_a1':
          'Bloom adalah alat bantu diri yang dirancang untuk membantu orang-orang mengurangi kecemasan sosial melalui proses yang disebut \'Paparan Bertahap\' (Graded Exposure). Dengan menyelesaikan tugas-tugas sosial yang kecil dan terukur, kamu melatih otakmu untuk menyadari bahwa interaksi sosial itu aman dan bisa dihadapi.',
      'faq_q2': 'Bagaimana cara kerja tingkatannya?',
      'faq_a2':
          'Kita mulai dari \'Bibit\' (tugas yang sangat mudah) dan naik perlahan hingga ke tahap \'Mekar\' (tugas yang lebih menantang). Setiap kali kamu menyelesaikan tugas, kamu akan mendapatkan poin kepercayaan diri. Semakin tinggi tingkatannya, semakin banyak poin yang kamu dapatkan!',
      'faq_q3': 'Apa itu Indikator Kepercayaan Diri (Confidence Meter)?',
      'faq_a3':
          'Indikator kemajuan di layar utamamu mewakili keseluruhan rasa percaya dirimu. Ini akan terus berkembang seiring kamu menyelesaikan tantangan. Namun hati-hati: jika kamu berhenti berlatih selama beberapa hari, skormu bisa sedikit menurun untuk mengingatkanmu bahwa kepercayaan diri adalah otot yang perlu dilatih secara teratur!',
      'faq_q4': 'Apa itu Rentetan Hari (Streak)?',
      'faq_a4':
          'Rentetan hari adalah hitungan berapa hari berturut-turut kamu menyelesaikan minimal satu tugas. Konsistensi adalah kunci utama untuk mengatasi kecemasan, jadi cobalah untuk menjaga apimu tetap menyala!',
      'faq_q5': 'Di mana data saya disimpan?',
      'faq_a5':
          'Privasimu adalah prioritas utama kami. Semua kemajuan, riwayat, dan data profilmu disimpan secara lokal di perangkatmu sendiri. Tidak ada data yang diunggah ke server cloud mana pun.',
      'faq_q6': 'Apa yang harus saya lakukan jika aplikasi eror (crash)?',
      'faq_a6':
          'Jika aplikasi berjalan tidak normal, cobalah mulai ulang ponselmu. Jika kamu baru saja memperbarui aplikasi, kamu mungkin perlu menghapus memori cache aplikasi di pengaturan Android-mu. Jika semua cara gagal, kamu bisa menggunakan pilihan \'Hapus Semua Kemajuan\' di halaman Profilmu.',
      'faq_q7': 'Apakah saya bisa melompati tingkatan?',
      'faq_a7':
          'Bisa! Meskipun kami menyarankan untuk mengikuti jalur bertahap, kamu sepenuhnya bebas memilih tingkatan mana saja dari peta yang terasa paling nyaman untuk kondisimu saat ini.',
      'faq_q8': 'Bagaimana jika tugasnya terlalu sulit?',
      'faq_a8':
          'Meskipun kami menyarankanmu untuk mencoba menyelesaikan tugas tersebut, jika itu benar-benar terlalu sulit, kamu cukup kembali ke layar utama lalu masuk lagi untuk mengganti tugas saat ini.',
      'faq_q9': 'Hubungi Kami',
      'faq_a9':
          'Kami sangat senang mendengar masukan dari pengguna mengenai aplikasi kami, termasuk saran untuk pembaruan di masa mendatang. Kami ingin tahu seberapa baik aplikasi ini membantumu, apa yang masih kurang, dan bagian mana yang perlu diperbaiki. Jangan ragu untuk membagikan saranmu di feedback.bloom@gmail.com, kami akan sangat menghargainya.',
      // Task Screen
      'stage_label': 'Tahap: {0}',
      'keep_growing': 'Teruslah berkembang, {0}',
      'current_challenge': 'Tantangan saat ini untuk tahap ini:',
      'stage_mastered': 'Tahap Berhasil Dikuasai!',
      'all_done': 'Kamu telah menyelesaikan semua tantangan di tahap ini.',
      'return_map': 'Kembali ke Peta',
      'well_done': 'Kerja bagus!',
      'i_completed': 'Saya Telah Menyelesaikan Ini',
      'level_up_suggestion_title': 'Saran Naik Tingkat',
      'level_up_suggestion_message':
          'Kamu telah menyelesaikan 10 tugas di tingkat ini! Kamu sudah siap untuk tingkat berikutnya. Ingin naik tingkat sekarang?',
      'stay_here': 'Tetap di Sini Dulu',
      'move_to_next_level': 'Pindah ke Tingkat Berikutnya',
      // Reflection Screen
      'reflect': 'Refleksikan perkembanganmu',
      'challenge': 'Tantangan',
      'anxiety_q': 'Seberapa cemas yang kamu rasakan? (1-10)',
      'what_happened': 'Apa yang sebenarnya terjadi?',
      'write_experience_hint': 'Tulis tentang pengalamanmu...',
      'finish': 'Selesai Refleksi',
      // General / Auth
      'welcome': 'Selamat Datang di Bloom',
      'subtitle': 'Ruang aman untuk menumbuhkan rasa percaya dirimu.',
      'start': 'Mulai Perjalananku',
      'guest': 'Lanjutkan sebagai Tamu',
      'hello': 'Halo',
      'profile': 'Profil Saya',
      'history': 'Perjalanan Perkembangan',
      'streak': 'Rentetan Hari Ini',
      'best': 'Rentetan Terbaik',
      'points': 'Poin Kepercayaan Diri',
      //Splash Screen
      'loading': 'Memuat tamanmu...',
    },
    'ga': {
      // Progress Screen
      'your_journey': 'Do Thuras',
      'keep_growing_sub': 'Bua is ea gach céim bheag. Lean ort ag fás!',
      'how_it_works': 'Conas a oibríonn sé?',
      'total_points': 'Pointí Iomlána',
      'current_streak': 'Rith Reatha',
      'tasks_done': 'Tascanna Críochnaithe',
      'rank': 'Céim',
      'days': 'Laethanta',
      'contact_us': 'Teagmháil a dhéanamh linn',
      'contact_email_prompt':
          'Le haghaidh tacaíochta agus aiseolais, seol ríomhphost chugainn ag:',
      'close': 'Dún',
      // Level Map Screen
      'tap_to_view_journey': 'Tapáil chun do thuras a fheiceáil! 🌸',
      'tap_to_start': 'Tapáil chun an dúshlán a thosú',
      'choose_level': 'Roghnaigh do chéim fháis:',
      'view_journey': 'Féach ar an Turas Fáis',
      'progress': 'Do Dhul Chun Cinn Fáis',
      'level_seedling': 'Síolán (Seedling)',
      'level_sprout': 'Péacán (Sprout)',
      'level_leaf': 'Duilleog (Leaf)',
      'level_stem': 'Gas (Stem)',
      'level_bloom': 'Bláthú (Bloom)',
      // Profile Screen
      'account': 'Cuntas',
      'display_name': 'Ainm Taispeána',
      'save_name': 'Sábháil an tAinm',
      'app_theme': 'Téama an Fheidhmchláir',
      'select_color': 'Roghnaigh do Dhath Bloom:',
      'light_mode': 'Solas',
      'dark_mode': 'Dorchadas',
      'language': 'Teanga',
      'logout': 'Logáil Amach',
      'profile_updated': 'Próifíl nuashonraithe!',
      'pick_theme_color': 'Roghnaigh dath téama',
      'done': 'Déanta',
      'reset_all_progress': 'Athshocraigh Gach Dul Chun Cinn',
      'reset_confirm_title': 'An bhfuil tú cinnte?',
      'reset_confirm_message':
          'Scriosfaidh sé seo do scór muiníne, do rith, agus do stair ar fad go buan. Ní féidir é seo a chur ar ceal.',
      'cancel': 'Cealaigh',
      'reset_everything': 'Athshocraigh Gach Rud',
      'reset_success': 'Athshocraíodh gach dul chun cinn.',
      // Shop Screen
      'shop_title': 'Siopa Bloom',
      'your_points': 'Do Phointí',
      'available_items': 'Míreanna Atá Ar Fáil',
      'streak_freeze': 'Reo Rithe',
      'protects_streak': 'Cosnaíonn sé do rith ó athshocrú',
      'your_inventory': 'Do Fhardal',
      'owned': 'In úinéireacht',
      'equipped': 'Feistithe',
      'equip_freeze': 'Feistigh Reo',
      'buy': 'Ceannaigh',
      'freeze_purchased': 'Reo Ceannaithe!',
      'not_enough_points': 'Níl go leor pointí agat!',
      'freeze_equipped': 'Reo feistithe!',
      // Milestone Screen
      'your_growth_path': 'Do Chonair Fáis',
      'the_awakening': 'An Dúiseacht',
      'seed_badge': 'Suaitheantas Síl',
      'first_spark': 'An Chéad Splanc',
      'bronze_leaf': 'Duilleog Chré-umha',
      'social_courage': 'Misneach Sóisialta',
      'silver_sprout': 'Péacán Airgid',
      'confidence_bloom': 'Bláthú Muiníne',
      'gold_flower': 'Bláth Óir',
      'mastery': 'Máistreacht',
      'diamond_crown': 'Coróin Diamaint',
      'milestone_claimed': 'Éilithe',
      'milestone_need_score': 'Ag teastáil {0}%',
      'milestone_claim_reward': 'Éiligh +{0} ptí',
      'milestone_reward_toast': 'Éilithe! +{0} ptí',
      // FAQ Screen
      'help_faq': 'Cabhair agus CCanna',
      'common_questions': 'Ceisteanna Coitianta',
      'keep_blooming': 'Lean ort ag bláthú! 🌸',
      'faq_q1': 'Cad é Bloom?',
      'faq_a1':
          'Is uirlis féinchabhrach é Bloom atá deartha chun cabhrú le daoine imní shóisialta a laghdú trí phróiseas ar a dtugtar \'Nochtadh Céimnithe\' (Graded Exposure). Trí thascanna sóisialta beaga, inoibrithe a chur i gcrích, traenálann tú d\'inchinn chun a thuiscint go bhfuil idirghníomhaíochtaí sóisialta sábháilte agus soláimhsithe.',
      'faq_q2': 'Conas a oibríonn na leibhéil?',
      'faq_a2':
          'Tosaímid le \'Síolán\' (tascanna an-éasca) edus bogaimid suas go dtí \'Bláthú\' (tascanna níos dúshlánaí). De réir mar a chríochnaíonn tú tascanna, gnóthaíonn tú pointí muiníne. Dá airde an leibhéal, is ea is mó pointí a ghnóthaíonn tú!',
      'faq_q3': 'Cad é an Méadar Muiníne?',
      'faq_a3':
          'Léiríonn an barra dul chun cinn ar do scáileán baile do mhuinín ghinearálta. Fásann sé de réir mar a chríochnaíonn tú tascanna. Bí cúramach: má stopann tú de bheith ag cleachtadh ar feadh roinnt laethanta, seans go laghdóidh do scór muiníne beagán, ag meabhrú duit gur matán é an mhuinín a dteastaíonn cleachtadh rialta uaidh!',
      'faq_q4': 'Cad é Rith (Streak)?',
      'faq_a4':
          'Is éard is rith ann ná comhaireamh ar cé mhéad lá as a chéile a chríochnaigh tú tasc amháin ar a laghad. Is é an comhsheasmhacht an eochair chun imní a shárú, mar sin déan iarracht do lasair a choinneáil ar lasadh!',
      'faq_q5': 'Cá bhfuil mo chuid sonraí stóráilte?',
      'faq_a5':
          'Is é do phríobháideachas ár bpríomhthosaíocht. Stóráiltear do dhul chun cinn, do stair agus do shonraí próifíle go léir go háitiúil ar do ghléas féin. Ní dhéantar aon rud a uaslódáil chuig freastalaí scamallach.',
      'faq_q6': 'Cad a dhéanfaidh mé má thuairteann an feidhmchlár?',
      'faq_a6':
          'Má bhíonn an feidhmchlár ag iompar go haisteach, déan iarracht do ghuthán a atosú. Má tá an feidhmchlár nuashonraithe agat, seans go mbeidh ort taisce an fheidhmchláir (cache) a ghlanadh i do shocruithe Android. Mura n-éiríonn le haon rud eile, is féidir leat an rogha \'Athshocraigh Gach Dul Chun Cinn\' a úsáid i do Phróifíl.',
      'faq_q7': 'An féidir liom leibhéil a scipeáil?',
      'faq_a7':
          'Is féidir! Cé go molaimid an chonair chéimnitheach, tá tú saor chun aon leibhéal a roghnú ón léarscáil a bhraitheann tú atá oiriúnach do do leibhéal compoird reatha.',
      'faq_q8': 'Cad tarlóidh má tá an tasc an-deacair?',
      'faq_a8':
          'Cé go molaimid duit iarracht a dhéanamh an tasc a chur i gcrích, is féidir leat dul ar ais go dtí an scáileán baile, agus dul isteach arís chun an tasc reatha a athrú.',
      'faq_q9': 'Teagmháil a dhéanamh linn',
      'faq_a9':
          'Ba bhreá linn tuairimí a chloisteáil faoinár bhfeidhmchlár ónár n-úsáideoirí, chomh maith le moltaí maidir le nuashonruithe amach anseo. Ba bhreá linn a chloisteáil cé chomh maith agus a oibríonn an feidhmchlár d\'úsáideoirí, cad atá in easnamh air agus cad a dteastaíonn feabhsú uaidh. Ná bíodh drogall ort aiseolas a roinnt ar feedback.bloom@gmail.com, bheimis an-bhuíoch as.',
      // Task Screen
      'stage_label': 'Céim: {0}',
      'keep_growing': 'Lean ort ag fás, {0}',
      'current_challenge': 'Do dhúshlán reatha don chéim seo:',
      'stage_mastered': 'Céim Máistrithe!',
      'all_done': 'Tá gach dúshlán sa chéim seo curtha i gcrích agat.',
      'return_map': 'Fill ar an Léarscáil',
      'well_done': 'Maith thú!',
      'i_completed': 'Chríochnaigh mé É Seo',
      'level_up_suggestion_title': 'Moladh Leibhéal Airde',
      'level_up_suggestion_message':
          'Tá 10 tasc curtha i gcrích agat ar an leibhéal seo! Tá tú réidh don chéad leibhéal eile. Ar mhaith leat bogadh suas?',
      'stay_here': 'Fan Anseo',
      'move_to_next_level': 'Bog go dtí an Chéad Leibhéal Eile',
      // Reflection Screen
      'reflect': 'Machnaigh ar do fhás',
      'challenge': 'Dúshlán',
      'anxiety_q': 'Cé chomh himníoch as a bhraith tú? (1-10)',
      'what_happened': 'Cad a tharla i ndáiríre?',
      'write_experience_hint': 'Scríobh faoi do thaithí...',
      'finish': 'Críochnaigh an Machnamh',
      // General / Auth
      'welcome': 'Fáilte go Bloom',
      'subtitle': 'Spás sábháilte chun do mhuinín a fhás.',
      'start': 'Tosaigh Mo Thuras',
      'guest': 'Lean ort mar Aoi',
      'hello': 'Dia duit',
      'profile': 'Mo Phróifíl',
      'history': 'Mo Thuras Fáis',
      'streak': 'Rith Reatha',
      'best': 'An Rith is Fearr',
      'points': 'Pointí Muiníne',
      //Splash Screen
      'loading': 'Ag lódáil do ghairdín...',
    },
    'sv': {
      // Progress Screen
      'your_journey': 'Din resa',
      'keep_growing_sub': 'Varje litet steg är en seger. Fortsätt växa!',
      'how_it_works': 'Hur fungerar det?',
      'total_points': 'Totala poäng',
      'current_streak': 'Nuvarande svit',
      'tasks_done': 'Slutförda uppgifter',
      'rank': 'Rang',
      'days': 'Dagar',
      'contact_us': 'Kontakta oss',
      'contact_email_prompt': 'För support och feedback, mejla oss på:',
      'close': 'Stäng',
      // Level Map Screen
      'tap_to_view_journey': 'Tryck för att se din resa! 🌸',
      'tap_to_start': 'Tryck för att starta utmaningen',
      'choose_level': 'Välj ditt tillväxtstadium:',
      'view_journey': 'Visa tillväxtresa',
      'progress': 'Dina tillväxtframsteg',
      'level_seedling': 'Grodd (Seedling)',
      'level_sprout': 'Skott (Sprout)',
      'level_leaf': 'Blad (Leaf)',
      'level_stem': 'Stjälk (Stem)',
      'level_bloom': 'Blomstring (Bloom)',
      // Profile Screen
      'account': 'Konto',
      'display_name': 'Visningsnamn',
      'save_name': 'Spara namn',
      'app_theme': 'Apptema',
      'select_color': 'Välj din Bloom-färg:',
      'light_mode': 'Ljust',
      'dark_mode': 'Mörkt',
      'language': 'Språk',
      'logout': 'Logga ut',
      'profile_updated': 'Profilen har uppdaterats!',
      'pick_theme_color': 'Välj en temafärg',
      'done': 'Klar',
      'reset_all_progress': 'Nollställ alla framsteg',
      'reset_confirm_title': 'Är du säker?',
      'reset_confirm_message':
          'Detta kommer att permanent radera dina självförtroendepoäng, din svit och all historik. Detta kan inte ångras.',
      'cancel': 'Avbryt',
      'reset_everything': 'Nollställ allt',
      'reset_success': 'Alla framsteg har nollställts.',
      // Shop Screen
      'shop_title': 'Bloom-butik',
      'your_points': 'Dine poäng',
      'available_items': 'Tillgängliga föremål',
      'streak_freeze': 'Svit-frys',
      'protects_streak': 'Skyddar din svit från att nollställas',
      'your_inventory': 'Ditt förråd',
      'owned': 'Ägs',
      'equipped': 'Aktiv',
      'equip_freeze': 'Aktivera frys',
      'buy': 'Köp',
      'freeze_purchased': 'Svit-frys köpt!',
      'not_enough_points': 'Inte tillräckligt med poäng!',
      'freeze_equipped': 'Frys aktiverad!',
      // Milestone Screen
      'your_growth_path': 'Din tillväxtväg',
      'the_awakening': 'Uppvaknandet',
      'seed_badge': 'Frö-märke',
      'first_spark': 'Första gnistan',
      'bronze_leaf': 'Bronsblad',
      'social_courage': 'Socialt mod',
      'silver_sprout': 'Silverskott',
      'confidence_bloom': 'Självförtroendeblomstring',
      'gold_flower': 'Guldblomma',
      'mastery': 'Mästare',
      'diamond_crown': 'Diamantkrona',
      'milestone_claimed': 'Hämtad',
      'milestone_need_score': 'Behöver {0}%',
      'milestone_claim_reward': 'Hämta +{0} poäng',
      'milestone_reward_toast': 'Hämtad! +{0} poäng',
      // FAQ Screen
      'help_faq': 'Hjälp och vanliga frågor',
      'common_questions': 'Vanliga frågor',
      'keep_blooming': 'Fortsätt blomstra! 🌸',
      'faq_q1': 'Vad är Bloom?',
      'faq_a1':
          'Bloom är ett självhjälpsverktyg designat för att hjälpa människor att minska social ångest genom en process som kallas \'Gradvis exponering\' (Graded Exposure). Genom att slutföra små, hanterbara sociala uppgifter tränar du din hjärna att inse att sociala interaktioner är säkra och hanterbara.',
      'faq_q2': 'Hur fungerar nivåerna?',
      'faq_a2':
          'Vi börjar med \'Grodd\' (väldigt enkla uppgifter) och rör oss upp till \'Blomstring\' (mer utmanande uppgifter). Allteftersom du slutför uppgifter tjänar du självförtroendepoäng. Ju högre nivån är, desto fler poäng tjänar du!',
      'faq_q3': 'Vad är självförtroendemätaren?',
      'faq_a3':
          'Förloppsindikatorn på din startskärm representerar ditt övergripande självförtroende. Den växer när du slutför uppgifter. Var försiktig: om du slutar öva under flera dagar kan dina självförtroendepoäng sjunka något, vilket påminner dig om att självförtroende är en muskel som behöver regelbunden träning!',
      'faq_q4': 'Vad är en svit (Streak)?',
      'faq_a4':
          'En svit är räkningen av hur många dagar i rad du har slutfört minst en uppgift. Konsistens är nyckeln till att övervinna ångest, så se till att hålla din flamma levande!',
      'faq_q5': 'Var lagras mina data?',
      'faq_a5':
          'Din integritet är vår prioritet. Alla dina framsteg, din historik och dina profildata lagras lokalt på din egen enhet. Ingenting laddas upp till någon molnserver.',
      'faq_q6': 'Vad gör jag om appen kraschar?',
      'faq_a6':
          'Om appen beter sig märkligt, prova att starta om din telefon. Om du har uppdaterat appen kan du behöva rensa appens cache i dina Android-inställningar. Om allt annat misslyckas kan du använda alternativet \'Nollställ alla framsteg\' under din profil.',
      'faq_q7': 'Kan jag hoppa över nivåer?',
      'faq_a7':
          'Ja! Även om vi rekommenderar den gradvisa vägen, är du fri att välja vilken nivå som helst från kartan som känns lämplig för din nuvarande komfortnivå.',
      'faq_q8': 'Vad om uppgiften är väldigt svår?',
      'faq_a8':
          'Även om vi rekommenderar att du försöker slutföra uppgiften, kan du bara gå tillbaka till startskärmen och öppna igen för att byta den nuvarande uppgiften.',
      'faq_q9': 'Kontakta oss',
      'faq_a9':
          'Vi vill jättegärna höra vad våra användare tycker om vår app, samt ta emot rekommendationer för framtida uppdateringar. Vi vill gärna veta hur väl appen fungerar för användarna, vad den saknar och vad som kräver förbättring. Dela gärna feedback på feedback.bloom@gmail.com, det skulle vi verkligen uppskatta.',
      // Task Screen
      'stage_label': 'Etapp: {0}',
      'keep_growing': 'Fortsätt växa, {0}',
      'current_challenge': 'Din nuvarande utmaning för denna etapp:',
      'stage_mastered': 'Etapp bemästrad!',
      'all_done': 'Du har slutfört alla utmaningar i denna etapp.',
      'return_map': 'Tillbaka till kartan',
      'well_done': 'Bra gjort!',
      'i_completed': 'Jag har slutfört detta',
      'level_up_suggestion_title': 'Förslag om nivåhöjning',
      'level_up_suggestion_message':
          'Du har slutfört 10 uppgifter på den här nivån! Du är redo för nästa nivå. Vill du gå upp?',
      'stay_here': 'Stanna kvar här',
      'move_to_next_level': 'Gå till nästa nivå',
      // Reflection Screen
      'reflect': 'Reflektera över din tillväxt',
      'challenge': 'Utmaning',
      'anxiety_q': 'Hur mycket ångest kände du? (1-10)',
      'what_happened': 'Vad hände egentligen?',
      'write_experience_hint': 'Skriv om din upplevelse...',
      'finish': 'Slutför reflektion',
      // General / Auth
      'welcome': 'Välkommen till Bloom',
      'subtitle': 'En säker plats att odla ditt självförtroende på.',
      'start': 'Starta min resa',
      'guest': 'Fortsätt som gäst',
      'hello': 'Hej',
      'profile': 'Min profil',
      'history': 'Min tillväxtresa',
      'streak': 'Nuvarande svit',
      'best': 'Bästa svit',
      'points': 'Självförtroendepoäng',
      //Splash Screen
      'loading': 'Lastar din trädgård...',
    },
  };

  static String get(String key, String lang) {
    final langMap = translations[lang];
    if (langMap != null && langMap.containsKey(key)) return langMap[key]!;
    final enMap = translations['en'];
    if (enMap != null && enMap.containsKey(key)) return enMap[key]!;
    debugPrint("⚠️ Translation MISSING: lang='$lang', key='$key'");
    return "[$key]";
  }

  // static String get(String key, String lang) =>
  //     translations[lang]?[key] ?? translations['en']![key]!;
}

// --- MILESTONE DEFINITION (NEW) ---
class MilestoneDefinition {
  final String id;
  final String titleKey;
  final String rewardKey;
  final IconData icon;
  final int requiredScore; // 0-100
  final int bonusPoints;
  final Color color;

  const MilestoneDefinition({
    required this.id,
    required this.titleKey,
    required this.rewardKey,
    required this.icon,
    required this.requiredScore,
    required this.bonusPoints,
    required this.color,
  });
}

// --- TASK LIBRARY ---
class TaskLibrary {
  static final Map<String, Map<String, List<Map<String, String>>>>
  translations = {
    'en': {
      "Seedling": [
        {
          "id": "S1",
          "title": "The First Step",
          "desc": "Make eye contact and smile at one person today.",
        },
        {
          "id": "S2",
          "title": "A Simple Hello",
          "desc": "Say 'Good morning' or 'Hello' to a neighbor.",
        },
        {
          "id": "S3",
          "title": "The Thank You",
          "desc": "Say 'Thank you' clearly to a shopkeeper.",
        },
        {
          "id": "S4",
          "title": "The Observation",
          "desc": "Notice something positive about a stranger and smile.",
        },
        {
          "id": "S5",
          "title": "The Quiet Wave",
          "desc": "Wave at someone you recognize from a distance.",
        },
        {
          "id": "S6",
          "title": "Door Hold",
          "desc": "Hold the door open for someone behind you.",
        },
        {
          "id": "S7",
          "title": "The Nod",
          "desc": "Give a friendly nod to a colleague as you pass them.",
        },
        {
          "id": "S8",
          "title": "The Mirror",
          "desc": "Practice your 'confident smile' in the mirror for 1 minute.",
        },
        {
          "id": "S9",
          "title": "The Brief Glance",
          "desc": "Look at someone for 2 seconds, then smile and look away.",
        },
        {
          "id": "S10",
          "title": "The Quiet Praise",
          "desc": "Write a nice comment on someone's social media post.",
        },
        {
          "id": "S11",
          "title": "The Space Share",
          "desc":
              "Sit next to someone in a public area without looking away immediately.",
        },
        {
          "id": "S12",
          "title": "The Simple Acknowledgement",
          "desc": "Say 'Excuse me' politely when passing someone in a hallway.",
        },
        {
          "id": "S13",
          "title": "The Warm Greeting",
          "desc": "Say 'Hi' to a delivery driver or courier.",
        },
        {
          "id": "S14",
          "title": "The Small Wave",
          "desc": "Wave to a child or a pet (with owner's permission).",
        },
        {
          "id": "S15",
          "title": "The Soft Smile",
          "desc": "Smile at three different people today.",
        },
        {
          "id": "S16",
          "title": "The Eye-Contact Challenge",
          "desc":
              "Maintain eye contact with a cashier until they look away first.",
        },
        {
          "id": "S17",
          "title": "The Gentle Breath",
          "desc": "Take 3 deep breaths before entering a social space today.",
        },
        {
          "id": "S18",
          "title": "The Presence",
          "desc":
              "Stand in a crowded area for 5 minutes without looking at your phone.",
        },
        {
          "id": "S19",
          "title": "The Casual Nod",
          "desc": "Nod to a stranger who makes eye contact with you.",
        },
        {
          "id": "S20",
          "title": "The Soft Voice",
          "desc": "Say 'Have a nice day' to someone as you leave a store.",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "The Compliment",
          "desc": "Give a genuine compliment to a colleague or classmate.",
        },
        {
          "id": "SP2",
          "title": "The Question",
          "desc": "Ask a stranger for the time or directions.",
        },
        {
          "id": "SP3",
          "title": "Small Talk",
          "desc":
              "Ask someone 'How is your day going?' and listen to the answer.",
        },
        {
          "id": "SP4",
          "title": "The Request",
          "desc": "Ask a store employee for help finding a specific item.",
        },
        {
          "id": "SP5",
          "title": "The Order",
          "desc": "Order a drink or food and ask the staff how they are doing.",
        },
        {
          "id": "SP6",
          "title": "The Greeting",
          "desc": "Introduce yourself to someone new in your area.",
        },
        {
          "id": "SP7",
          "title": "The Weather Talk",
          "desc": "Mention the weather to someone while waiting in a line.",
        },
        {
          "id": "SP8",
          "title": "The Simple Inquiry",
          "desc": "Ask a coworker 'What did you do over the weekend?'",
        },
        {
          "id": "SP9",
          "title": "The Help Offer",
          "desc":
              "Ask someone 'Do you need help with that?' if they look struggling.",
        },
        {
          "id": "SP10",
          "title": "The Opinion",
          "desc":
              "Ask a friend 'What do you think of this?' about a small object.",
        },
        {
          "id": "SP11",
          "title": "The Confirmation",
          "desc":
              "Confirm a detail with a stranger (e.g., 'Is this the right line?').",
        },
        {
          "id": "SP12",
          "title": "The Shared Space",
          "desc":
              "Make a small comment about the environment (e.g., 'It's really crowded').",
        },
        {
          "id": "SP13",
          "title": "The Small Favor",
          "desc":
              "Ask someone to pass you something (like a napkin) at a table.",
        },
        {
          "id": "SP14",
          "title": "The Warm Feedback",
          "desc": "Tell a waiter that the food was great before leaving.",
        },
        {
          "id": "SP15",
          "title": "The Casual Check-in",
          "desc":
              "Send a 'How are you?' text to someone you haven't spoken to in a month.",
        },
        {
          "id": "SP16",
          "title": "The Open Question",
          "desc":
              "Ask someone 'Where is your favorite place to visit in this city?'",
        },
        {
          "id": "SP17",
          "title": "The Smallest Risk",
          "desc": "Ask a stranger if they know where the nearest restroom is.",
        },
        {
          "id": "SP18",
          "title": "The Item Praise",
          "desc": "Tell someone you like their shoes/bag/accessory.",
        },
        {
          "id": "SP19",
          "title": "The Polite Pause",
          "desc":
              "Wait for someone to finish speaking entirely before responding to them.",
        },
        {
          "id": "SP20",
          "title": "The Friendly Wave",
          "desc":
              "Wave and say 'Bye' to someone you just had a short interaction with.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "Opinion Seeker",
          "desc": "Ask someone for their opinion on a book, movie, or song.",
        },
        {
          "id": "L2",
          "title": "The Detail",
          "desc":
              "Ask a follow-up question after someone tells you something about themselves.",
        },
        {
          "id": "L3",
          "title": "The Recommendation",
          "desc":
              "Ask a stranger for a recommendation for a good place to eat nearby.",
        },
        {
          "id": "L4",
          "title": "The Connection",
          "desc":
              "Find a common interest with someone and talk about it for 2 minutes.",
        },
        {
          "id": "L5",
          "title": "The Helpful Hand",
          "desc":
              "Offer to help someone with a small task (like carrying a bag).",
        },
        {
          "id": "L6",
          "title": "The Social Observation",
          "desc":
              "Start a conversation based on something happening around you both.",
        },
        {
          "id": "L7",
          "title": "The Open Ended Question",
          "desc": "Ask someone 'How did you get into this line of work?'",
        },
        {
          "id": "L8",
          "title": "The Active Listener",
          "desc":
              "Listen to someone for 3 minutes without interrupting, then summarize what they said.",
        },
        {
          "id": "L9",
          "title": "The Shared Laugh",
          "desc": "Tell a short, funny story or a joke to a small group.",
        },
        {
          "id": "L10",
          "title": "The Curiosity",
          "desc":
              "Ask someone where they are from and what they like about that place.",
        },
        {
          "id": "L11",
          "title": "The Sincere Interest",
          "desc": "Ask a colleague about their hobbies outside of work.",
        },
        {
          "id": "L12",
          "title": "The Soft Advice",
          "desc": "Give someone a helpful tip on something you are good at.",
        },
        {
          "id": "L13",
          "title": "The Group Nod",
          "desc": "Agree with someone's point in a small group discussion.",
        },
        {
          "id": "L14",
          "title": "The Casual Invitation",
          "desc": "Ask someone 'Would you like to join us for lunch?'",
        },
        {
          "id": "L15",
          "title": "The Honest Reflection",
          "desc":
              "Tell someone 'I really appreciated it when you did X' and explain why.",
        },
        {
          "id": "L16",
          "title": "The Curiosity Gap",
          "desc":
              "Ask someone 'I've always wondered, how does X actually work?'",
        },
        {
          "id": "L17",
          "title": "The Small Group Lead",
          "desc":
              "Ask a question that requires 2 or 3 people in a group to answer.",
        },
        {
          "id": "L18",
          "title": "The Genuine Compliment",
          "desc":
              "Compliment someone on a personality trait (e.g., 'You're a great listener').",
        },
        {
          "id": "L19",
          "title": "The Shared Experience",
          "desc":
              "Say 'I've been in that situation too' during a conversation.",
        },
        {
          "id": "L20",
          "title": "The Meaningful Pause",
          "desc":
              "Allow a silence to happen in a conversation without rushing to fill it.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "The Brave Start",
          "desc": "Start a conversation with someone you don't know well.",
        },
        {
          "id": "ST2",
          "title": "The Honest Share",
          "desc": "Share a small personal story or opinion in a group setting.",
        },
        {
          "id": "ST3",
          "title": "The Debate",
          "desc": "Politely disagree with someone's opinion and explain why.",
        },
        {
          "id": "ST4",
          "title": "The Group Entry",
          "desc":
              "Join a group conversation and contribute a thoughtful sentence.",
        },
        {
          "id": "ST4",
          "title": "The Topic Lead",
          "desc": "Bring up a new topic of conversation in a social group.",
        },
        {
          "id": "ST6",
          "title": "The Public Question",
          "desc": "Ask a question in a public meeting or a classroom setting.",
        },
        {
          "id": "ST7",
          "title": "The Bold Request",
          "desc":
              "Ask a stranger if you can sit next to them at a cafe or park.",
        },
        {
          "id": "ST8",
          "title": "The Conversation Bridge",
          "desc":
              "Introduce two people who don't know each other and find a commonality.",
        },
        {
          "id": "ST9",
          "title": "The Assertive Need",
          "desc":
              "Politely ask someone to move or stop doing something that bothers you.",
        },
        {
          "id": "ST10",
          "title": "The Storyteller",
          "desc":
              "Take the lead in telling a story to a group of 3 or more people.",
        },
        {
          "id": "ST11",
          "title": "The Open Challenge",
          "desc":
              "Challenge a common opinion in a group in a friendly, respectful way.",
        },
        {
          "id": "ST12",
          "title": "The Social Initiative",
          "desc":
              "Be the first person to say 'Hello' to everyone when entering a room.",
        },
        {
          "id": "ST13",
          "title": "The Empathetic Listen",
          "desc":
              "Listen to someone venting and provide a supportive response.",
        },
        {
          "id": "ST14",
          "title": "The Public Presentation",
          "desc":
              "Speak for 1-2 minutes about a topic you love in a social gathering.",
        },
        {
          "id": "ST15",
          "title": "The Vulnerable Share",
          "desc":
              "Admit to a group that you were nervous about something, and laugh about it.",
        },
        {
          "id": "ST16",
          "title": "The Boundary Set",
          "desc":
              "Politely decline an invitation you don't want to attend without over-explaining.",
        },
        {
          "id": "ST17",
          "title": "The Active Mediator",
          "desc": "Help two people find a middle ground in a disagreement.",
        },
        {
          "id": "ST18",
          "title": "The Public Compliment",
          "desc": "Publicly praise someone's effort or achievement in a group.",
        },
        {
          "id": "ST19",
          "title": "The Direct Approach",
          "desc":
              "Ask someone directly for a favor or a piece of advice you need.",
        },
        {
          "id": "ST20",
          "title": "The Conversation Pivot",
          "desc":
              "Smoothly transition a conversation from a boring topic to an interesting one.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "The Gift",
          "desc":
              "Give a small treat to someone and say 'I thought you'd like this'.",
        },
        {
          "id": "B2",
          "title": "The Bold Lead",
          "desc":
              "Suggest a plan or a place to visit to a small group of people.",
        },
        {
          "id": "B3",
          "title": "The Appreciation",
          "desc":
              "Tell someone specifically why you appreciate having them in your life.",
        },
        {
          "id": "B4",
          "title": "The Social Host",
          "desc":
              "Organize a small get-together or a coffee date for a few people.",
        },
        {
          "id": "B5",
          "title": "The Deep Dive",
          "desc":
              "Have a deep, meaningful conversation with someone for over 15 minutes.",
        },
        {
          "id": "B6",
          "title": "The Confidence Peak",
          "desc": "Initiate a conversation with someone you find intimidating.",
        },
        {
          "id": "B7",
          "title": "The Public Toast",
          "desc":
              "Make a short, positive toast or shout-out to someone in a group.",
        },
        {
          "id": "B8",
          "title": "The Boundary Setter",
          "desc":
              "Say 'No' to a request firmly but kindly, without over-explaining.",
        },
        {
          "id": "B9",
          "title": "The Direct Request",
          "desc": "Ask someone you admire for a 10-minute chat or mentorship.",
        },
        {
          "id": "B10",
          "title": "The Emotional Lead",
          "desc":
              "Initiate a conversation about feelings or mental health with a friend.",
        },
        {
          "id": "B11",
          "title": "The Social Mediator",
          "desc":
              "Help two people resolve a small conflict through a calm conversation.",
        },
        {
          "id": "B12",
          "title": "The Bold Compliment",
          "desc":
              "Tell a complete stranger something you genuinely admire about them.",
        },
        {
          "id": "B13",
          "title": "The Networking Move",
          "desc":
              "Introduce yourself to a professional in your field and ask for advice.",
        },
        {
          "id": "B14",
          "title": "The Courageous Truth",
          "desc":
              "Tell someone a truth that is difficult but helpful for the relationship.",
        },
        {
          "id": "B15",
          "title": "The Full Bloom",
          "desc":
              "Host a small social event and make sure every guest feels welcome.",
        },
        {
          "id": "B16",
          "title": "The Public Speaker",
          "desc":
              "Volunteer to speak or lead a small part of a meeting or event.",
        },
        {
          "id": "B17",
          "title": "The Vulnerable Lead",
          "desc": "Share a struggle you've overcome to encourage someone else.",
        },
        {
          "id": "B18",
          "title": "The Bold Apology",
          "desc":
              "Initiate a conversation to apologize for a past mistake, even if it was long ago.",
        },
        {
          "id": "B19",
          "title": "The Mentor",
          "desc":
              "Offer to help someone who is less experienced than you with a skill.",
        },
        {
          "id": "B20",
          "title": "The Social Architect",
          "desc":
              "Create a new social tradition or a recurring meetup for a group of friends.",
        },
      ],
    },
    'es': {
      "Seedling": [
        {
          "id": "S1",
          "title": "El Primer Paso",
          "desc": "Mantén contacto visual y sonríe a una persona hoy.",
        },
        {
          "id": "S2",
          "title": "Un Hola Simple",
          "desc": "Di 'Buenos días' o 'Hola' a un vecino.",
        },
        {
          "id": "S3",
          "title": "El Gracias",
          "desc": "Di 'Gracias' claramente a un tendero.",
        },
        {
          "id": "S4",
          "title": "La Observación",
          "desc": "Nota algo positivo sobre un extraño y sonríe.",
        },
        {
          "id": "S5",
          "title": "El Saludo Silencioso",
          "desc":
              "Saluuda con la mano a alguien que reconozcas a la distancia.",
        },
        {
          "id": "S6",
          "title": "Sostener la Puerta",
          "desc": "Sostén la puerta abierta para alguien detrás de ti.",
        },
        {
          "id": "S7",
          "title": "El Asentimiento",
          "desc": "Haz un gesto amable con la cabeza a un colega al pasar.",
        },
        {
          "id": "S8",
          "title": "El Espejo",
          "desc":
              "Practica tu 'sonrisa confiada' frente al espejo durante 1 minuto.",
        },
        {
          "id": "S9",
          "title": "La Mirada Breve",
          "desc":
              "Mira a alguien durante 2 segundos, luego sonríe y mira hacia otro lado.",
        },
        {
          "id": "S10",
          "title": "El Elogio Silencioso",
          "desc":
              "Escribe un comentario amable en la publicación de alguien en redes sociales.",
        },
        {
          "id": "S11",
          "title": "Compartir el Espacio",
          "desc":
              "Siéntate junto a alguien en un área pública sin apartar la mirada inmediatamente.",
        },
        {
          "id": "S12",
          "title": "El Reconocimiento Simple",
          "desc":
              "Di 'Disculpe' educadamente al pasar junto a alguien en un pasillo.",
        },
        {
          "id": "S13",
          "title": "El Saludo Cálido",
          "desc": "Di 'Hola' a un repartidor or mensajero.",
        },
        {
          "id": "S14",
          "title": "El Pequeño Saludo",
          "desc":
              "Saluuda con la mano a un niño or a una mascota (con permiso del dueño).",
        },
        {
          "id": "S15",
          "title": "La Sonrisa Suave",
          "desc": "Sonríe a tres personas diferentes hoy.",
        },
        {
          "id": "S16",
          "title": "El Reto del Contacto Visual",
          "desc":
              "Mantén el contacto visual con un cajero hasta que él mire hacia otro lado primero.",
        },
        {
          "id": "S17",
          "title": "La Respiración Suave",
          "desc":
              "Toma 3 respiraciones profundas antes de entrar en un espacio social hoy.",
        },
        {
          "id": "S18",
          "title": "La Presencia",
          "desc":
              "Quédate en un área concurrida durante 5 minutos sin mirar tu teléfono.",
        },
        {
          "id": "S19",
          "title": "El Asentimiento Casual",
          "desc":
              "Asiente con la cabeza a un extraño que haga contacto visual contigo.",
        },
        {
          "id": "S20",
          "title": "La Voz Suave",
          "desc":
              "Dile 'Que tenga un buen día' a alguien al salir de una tienda.",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "El Cumplido",
          "desc": "Dale un cumplido genuino a un colega or compañero de clase.",
        },
        {
          "id": "SP2",
          "title": "La Pregunta",
          "desc": "Pregunta a un extraño la hora or direcciones.",
        },
        {
          "id": "SP3",
          "title": "Charla Casual",
          "desc":
              "Pregunta a alguien '¿Cómo va tu día?' y escucha la respuesta.",
        },
        {
          "id": "SP4",
          "title": "La Petición",
          "desc":
              "Pide ayuda a un empleado de la tienda para encontrar un artículo específico.",
        },
        {
          "id": "SP5",
          "title": "El Pedido",
          "desc":
              "Pide una bebida or comida y pregunta al personal cómo están.",
        },
        {
          "id": "SP6",
          "title": "El Saludo",
          "desc": "Preséntate a alguien nuevo en tu área.",
        },
        {
          "id": "SP7",
          "title": "Charla sobre el Clima",
          "desc": "Menciona el clima a alguien mientras esperas en una fila.",
        },
        {
          "id": "SP8",
          "title": "La Consulta Simple",
          "desc": "Pregunta a un compañero '¿Qué hiciste el fin de semana?'",
        },
        {
          "id": "SP9",
          "title": "El Ofrecimiento de Ayuda",
          "desc":
              "Pregunta a alguien '¿Necesitas ayuda con eso?' si parece estar luchando.",
        },
        {
          "id": "SP10",
          "title": "La Opinión",
          "desc":
              "Pregunta a un amigo '¿Qué piensas de esto?' sobre un objeto pequeño.",
        },
        {
          "id": "SP11",
          "title": "La Confirmación",
          "desc":
              "Confirma un detalle con un extraño (ej., '¿Es esta la fila correcta?').",
        },
        {
          "id": "SP12",
          "title": "El Espacio Compartido",
          "desc":
              "Haz un comentario pequeño sobre el entorno (ej., 'Está muy lleno hoy').",
        },
        {
          "id": "SP13",
          "title": "El Pequeño Favor",
          "desc":
              "Pide a alguien que te pase algo (como una servilleta) en una mesa.",
        },
        {
          "id": "SP14",
          "title": "La Retroalimentación Cálida",
          "desc": "Dile a un mesero que la comida estuvo genial antes de irte.",
        },
        {
          "id": "SP15",
          "title": "El Saludo Casual",
          "desc":
              "Envía un texto de '¿Cómo estás?' a alguien con quien no has hablado en un mes.",
        },
        {
          "id": "SP16",
          "title": "La Pregunta Abierta",
          "desc":
              "Pregunta a alguien '¿Cuál es tu lugar favorito para visitar en esta ciudad?'",
        },
        {
          "id": "SP17",
          "title": "El Riesgo Más Pequeño",
          "desc":
              "Pregunta a un extraño si sabe dónde está el baño más cercano.",
        },
        {
          "id": "SP18",
          "title": "El Elogio al Objeto",
          "desc": "Dile a alguien que te gustan sus zapatos/bolso/accesorio.",
        },
        {
          "id": "SP19",
          "title": "La Pausa Educada",
          "desc":
              "Espera a que alguien termine de hablar completamente antes de responderle.",
        },
        {
          "id": "SP20",
          "title": "El Saludo Amistoso",
          "desc":
              "Saluuda con la mano y di 'Adiós' a alguien con quien acabas de tener una breve interacción.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "Buscador de Opiniones",
          "desc":
              "Pregunta a alguien su opinión sobre un libro, película or canción.",
        },
        {
          "id": "L2",
          "title": "El Detalle",
          "desc":
              "Haz una pregunta de seguimiento después de que alguien te cuente algo sobre sí mismo.",
        },
        {
          "id": "L3",
          "title": "La Recomendación",
          "desc":
              "Pregunta a un extraño por una recomendación de un buen lugar para comer cerca.",
        },
        {
          "id": "L4",
          "title": "La Conexión",
          "desc":
              "Encuentra un interés común con alguien y habla de ello durante 2 minutos.",
        },
        {
          "id": "L5",
          "title": "La Mano Ayudadora",
          "desc":
              "Ofrece ayuda a alguien con una tarea pequeña (como cargar una bolsa).",
        },
        {
          "id": "L6",
          "title": "La Observación Social",
          "desc":
              "Inicia una conversación basada en algo que esté sucediendo alrededor de ambos.",
        },
        {
          "id": "L7",
          "title": "La Pregunta Abierta",
          "desc": "Pregunta a alguien '¿Cómo llegaste a este campo laboral?'",
        },
        {
          "id": "L8",
          "title": "El Oyente Activo",
          "desc":
              "Escucha a alguien durante 3 minutos sin interrumpir, luego resume lo que dijo.",
        },
        {
          "id": "L9",
          "title": "La Risa Compartida",
          "desc":
              "Cuenta una historia corta y divertida or un chiste a un grupo pequeño.",
        },
        {
          "id": "L10",
          "title": "La Curiosidad",
          "desc": "Pregunta a alguien de dónde es y qué le gusta de ese lugar.",
        },
        {
          "id": "L11",
          "title": "El Interés Sincero",
          "desc":
              "Pregunta a un colega sobre sus pasatiempos fuera del trabajo.",
        },
        {
          "id": "L12",
          "title": "El Consejo Suave",
          "desc":
              "Dale a alguien un consejo útil sobre algo en lo que seas bueno.",
        },
        {
          "id": "L13",
          "title": "El Asentimiento Grupal",
          "desc":
              "Estar de acuerdo con el punto de alguien en una discusión de grupo pequeño.",
        },
        {
          "id": "L14",
          "title": "La Invitación Casual",
          "desc": "Pregunta a alguien '¿Te gustaría acompañarnos a almorzar?'",
        },
        {
          "id": "L15",
          "title": "La Reflexión Honesta",
          "desc":
              "Dile a alguien 'Realmente aprecié cuando hiciste X' y explica por qué.",
        },
        {
          "id": "L16",
          "title": "La Brecha de Curiosidad",
          "desc":
              "Pregunta a alguien 'Siempre me he preguntado, ¿cómo funciona X realmente?'",
        },
        {
          "id": "L17",
          "title": "Liderazgo de Grupo Pequeño",
          "desc":
              "Haz una pregunta que requiera que 2 or 3 personas de un grupo respondan.",
        },
        {
          "id": "L18",
          "title": "El Cumplido Genuino",
          "desc":
              "Elogia a alguien por un rasgo de su personalidad (ej., 'Eres un gran orador').",
        },
        {
          "id": "L19",
          "title": "La Experiencia Compartida",
          "desc":
              "Di 'Yo he estado en esa situación también' durante una conversación.",
        },
        {
          "id": "L20",
          "title": "La Pausa Significativa",
          "desc":
              "Permite que ocurra un silencio en una conversación sin apresurarse a llenarlo.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "El Inicio Valiente",
          "desc": "Inicia una conversación con alguien que no conozcas bien.",
        },
        {
          "id": "ST2",
          "title": "El Compartir Honesto",
          "desc":
              "Comparte una pequeña historia personal u opinión en un entorno grupal.",
        },
        {
          "id": "ST3",
          "title": "El Debate",
          "desc":
              "No estés de acuerdo educadamente con la opinión de alguien y explica por qué.",
        },
        {
          "id": "ST4",
          "title": "La Entrada al Grupo",
          "desc":
              "Únete a una conversación grupal y aporta una frase reflexiva.",
        },
        {
          "id": "ST5",
          "title": "El Líder del Tema",
          "desc": "Plantea un nuevo tema de conversación en un grupo social.",
        },
        {
          "id": "ST6",
          "title": "La Pregunta Pública",
          "desc":
              "Haz una pregunta en una reunión pública or en un salón de clases.",
        },
        {
          "id": "ST7",
          "title": "La Petición Audaz",
          "desc":
              "Pregunta a un extraño si puedes sentarte junto a él en un café or parque.",
        },
        {
          "id": "ST8",
          "title": "El Puente de Conversación",
          "desc":
              "Presenta a dos personas que no se conocen y encuentra algo en común.",
        },
        {
          "id": "ST9",
          "title": "La Necesidad Asertiva",
          "desc":
              "Pide educadamente a alguien que se mueva or deje de hacer algo que te moleste.",
        },
        {
          "id": "ST10",
          "title": "El Cuentacuentos",
          "desc":
              "Toma la iniciativa de contar una historia a un grupo de 3 or más personas.",
        },
        {
          "id": "ST11",
          "title": "El Desafío Abierto",
          "desc":
              "Cuestiona una opinión común en un grupo de manera amistosa y respetuosa.",
        },
        {
          "id": "ST12",
          "title": "La Iniciativa Social",
          "desc":
              "Sé la primera persona en decir 'Hola' a todos al entrar en una habitación.",
        },
        {
          "id": "ST13",
          "title": "La Escucha Empática",
          "desc":
              "Escucha a alguien desahogarse y proporciona una respuesta comprensiva.",
        },
        {
          "id": "ST14",
          "title": "La Presentación Pública",
          "desc":
              "Habla durante 1-2 minutos sobre un tema que ames en una reunión social.",
        },
        {
          "id": "ST15",
          "title": "El Compartir Vulnerable",
          "desc":
              "Admite ante un grupo que estabas nervioso por algo y ríete de ello.",
        },
        {
          "id": "ST16",
          "title": "El Límite Establecido",
          "desc":
              "Rechaza educadamente una invitación que no quieres aceptar sin dar demasiadas explicaciones.",
        },
        {
          "id": "ST17",
          "title": "El Mediador Activo",
          "desc":
              "Ayuda a dos personas a encontrar un punto medio en un desacuerdo.",
        },
        {
          "id": "ST18",
          "title": "El Cumplido Público",
          "desc":
              "Elogia públicamente el esfuerzo or logro de alguien en un grupo.",
        },
        {
          "id": "ST19",
          "title": "El Enfoque Directo",
          "desc":
              "Pide a alguien directamente un favor or un consejo que necesites.",
        },
        {
          "id": "ST20",
          "title": "El Pivote de Conversación",
          "desc":
              "Transiciona suavemente una conversación de un tema aburrido a uno interesante.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "El Regalo",
          "desc":
              "Dale un dulce a alguien y dile 'Pensé que te gustaría esto'.",
        },
        {
          "id": "B2",
          "title": "El Liderazgo Audaz",
          "desc":
              "Sugiere un plan or un lugar para visitar a un grupo pequeño de personas.",
        },
        {
          "id": "B3",
          "title": "El Aprecio",
          "desc":
              "Dile a alguien específicamente por qué aprecias tenerlo en tu vida.",
        },
        {
          "id": "B4",
          "title": "El Anfitrión Social",
          "desc":
              "Organiza una pequeña reunión or una cita para tomar café con algunas personas.",
        },
        {
          "id": "B5",
          "title": "La Inmersión Profunda",
          "desc":
              "Tén una conversación profunda y significativa con alguien durante más de 15 minutos.",
        },
        {
          "id": "B6",
          "title": "El Pico de Confianza",
          "desc":
              "Inicia una conversación con alguien que consideres intimidante.",
        },
        {
          "id": "B7",
          "title": "El Brindis Público",
          "desc":
              "Haz un brindis corto y positivo or un reconocimiento a alguien en un grupo.",
        },
        {
          "id": "B8",
          "title": "El Establecedor de Límites",
          "desc":
              "Di 'No' a una petición con firmeza pero amablemente, sin dar demasiadas explicaciones.",
        },
        {
          "id": "B9",
          "title": "La Petición Directa",
          "desc":
              "Pide a alguien que admires una charla de 10 minutos or mentoría.",
        },
        {
          "id": "B10",
          "title": "El Liderazgo Emocional",
          "desc":
              "Inicia una conversación sobre sentimientos or salud mental con un amigo.",
        },
        {
          "id": "B11",
          "title": "El Mediador Social",
          "desc":
              "Ayuda a dos personas a resolver un conflicto pequeño a través de una conversación calmada.",
        },
        {
          "id": "B12",
          "title": "El Cumplido Audaz",
          "desc":
              "Dile a un completo extraño algo que genuinamente admires de ellos.",
        },
        {
          "id": "B13",
          "title": "El Movimiento de Networking",
          "desc": "Preséntate a un profesional de tu campo y pide consejo.",
        },
        {
          "id": "B14",
          "title": "La Verdad Valiente",
          "desc":
              "Dile a alguien una verdad que sea difícil de decir, pero útil para la relación.",
        },
        {
          "id": "B15",
          "title": "El Florecimiento Total",
          "desc":
              "Organiza un evento social pequeño y asegúrate de que cada invitado se sienta bienvenido.",
        },
        {
          "id": "B16",
          "title": "El Orador Público",
          "desc":
              "Ofrécete para hablar or dirigir una pequeña parte de una reunión or evento.",
        },
        {
          "id": "B17",
          "title": "El Liderazgo Vulnerable",
          "desc":
              "Comparte una lucha que hayas superCidado para animar a alguien más.",
        },
        {
          "id": "B18",
          "title": "La Disculpa Audaz",
          "desc":
              "Inicia una conversación para disculparte por un error pasado, incluso si fue hace mucho tiempo.",
        },
        {
          "id": "B19",
          "title": "El Mentor",
          "desc":
              "Ofrece ayuda a alguien que tenga menos experiencia que tú en una habilidad.",
        },
        {
          "id": "B20",
          "title": "El Arquitecto Social",
          "desc":
              "Crea una nueva tradición social or una reunión recurrente para un grupo de amigos.",
        },
      ],
    },
    'fr': {
      "Seedling": [
        {
          "id": "S1",
          "title": "Le Premier Pas",
          "desc":
              "Établissez un contact visuel et souriez à une personne aujourd'hui.",
        },
        {
          "id": "S2",
          "title": "Un Bonjour Simple",
          "desc": "Dites 'Bonjour' à un voisin.",
        },
        {
          "id": "S3",
          "title": "Le Merci",
          "desc": "Dites 'Merci' clairement à un commerçant.",
        },
        {
          "id": "S4",
          "title": "L'Observation",
          "desc":
              "Remarquez quelque chose de positif chez un étranger et souriez.",
        },
        {
          "id": "S5",
          "title": "Le Salut Silencieux",
          "desc":
              "Faites un signe de la main à quelqu'un que vous reconnaissez au loin.",
        },
        {
          "id": "S6",
          "title": "Tenir la Porte",
          "desc": "Tenez la porte ouverte pour quelqu'un derrière vous.",
        },
        {
          "id": "S7",
          "title": "Le Hochement de Tête",
          "desc": "Faites un signe de tête amical à un collègue en passant.",
        },
        {
          "id": "S8",
          "title": "Le Miroir",
          "desc":
              "Pratiquez votre 'sourire confiant' dans le miroir pendant 1 minute.",
        },
        {
          "id": "S9",
          "title": "Le Regard Bref",
          "desc":
              "Regardez quelqu'un pendant 2 secondes, puis souriez et détournez le regard.",
        },
        {
          "id": "S10",
          "title": "L'Éloge Silencieux",
          "desc":
              "Écrivez un commentaire gentil sur la publication de quelqu'un sur les réseaux sociaux.",
        },
        {
          "id": "S11",
          "title": "Partager l'Espace",
          "desc":
              "Asseyez-vous à côté de quelqu'un dans un lieu public sans détourner le regard immédiatement.",
        },
        {
          "id": "S12",
          "title": "L'Héritage Simple",
          "desc":
              "Dites 'Excusez-moi' poliment en passant devant quelqu'un dans un couloir.",
        },
        {
          "id": "S13",
          "title": "Le Salut Chaleureux",
          "desc": "Dites 'Salut' à un livreur or un coursier.",
        },
        {
          "id": "S14",
          "title": "Le Petit Salut",
          "desc":
              "Faites un signe de la main à un enfant or à un animal (avec la permission du propriétaire).",
        },
        {
          "id": "S15",
          "title": "Le Sourire Doux",
          "desc": "Sourire à trois personnes différentes aujourd'hui.",
        },
        {
          "id": "S16",
          "title": "Le Défi du Contact Visuel",
          "desc":
              "Maintenez le contact visuel avec un caissier jusqu'à ce qu'il détourne le regard en premier.",
        },
        {
          "id": "S17",
          "title": "La Respiration Douce",
          "desc":
              "Prenez 3 respirations profondes avant d'entrer dans un espace social aujourd'hui.",
        },
        {
          "id": "S18",
          "title": "La Présence",
          "desc":
              "Tenez-vous dans un endroit bondé pendant 5 minutes sans regarder votre téléphone.",
        },
        {
          "id": "S19",
          "title": "Le Hochement Casual",
          "desc":
              "Faites un signe de tête à un étranger qui établit un contact visuel avec vous.",
        },
        {
          "id": "S20",
          "title": "La Voix Douce",
          "desc": "Dites 'Bonne journée' à quelqu'un en quittant un magasin.",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "Le Compliment",
          "desc":
              "Faites un compliment sincère à un collègue or un camarade de classe.",
        },
        {
          "id": "SP2",
          "title": "La Question",
          "desc": "Demandez l'heure or des directions à un étranger.",
        },
        {
          "id": "SP3",
          "title": "Conversation Banale",
          "desc":
              "Demandez à quelqu'un 'Comment se passe votre journée ?' et écoutez la réponse.",
        },
        {
          "id": "SP4",
          "title": "La Demande",
          "desc":
              "Demandez l'aide d'un employé de magasin pour trouver un article spécifique.",
        },
        {
          "id": "SP5",
          "title": "La Commande",
          "desc":
              "Commandez une boisson or de la nourriture et demandez au personnel comment ils vont.",
        },
        {
          "id": "SP6",
          "title": "La Salutation",
          "desc": "Présentez-vous à quelqu'un de nouveau dans votre quartier.",
        },
        {
          "id": "SP7",
          "title": "Parler du Temps",
          "desc": "Mentionnez le temps à quelqu'un en attendant dans une file.",
        },
        {
          "id": "SP8",
          "title": "L'Interrogation Simple",
          "desc": "Demandez à un collègue 'Qu'as-tu fait ce week-end ?'",
        },
        {
          "id": "SP9",
          "title": "L'Offre d'Aide",
          "desc":
              "Demandez à quelqu'un 'Avez-vous besoin d'aide ?' s'il semble être en difficulté.",
        },
        {
          "id": "SP10",
          "title": "L'Opinion",
          "desc": "Demandez or l'avis d'un ami sur un petit objet.",
        },
        {
          "id": "SP11",
          "title": "La Confirmation",
          "desc":
              "Confirmez un détail avec un étranger (ex: 'Est-ce la bonne file ?').",
        },
        {
          "id": "SP12",
          "title": "L'Espace Partagé",
          "desc":
              "Faites un petit commentaire sur l'environnement (ex: 'C'est vraiment bondé aujourd'hui').",
        },
        {
          "id": "SP13",
          "title": "Le Petit Service",
          "desc":
              "Demandez à quelqu'un de vous passer quelque chose (comme une serviette) à une table.",
        },
        {
          "id": "SP14",
          "title": "Le Retour Chaleureux",
          "desc":
              "Dites à un serveur que la nourriture était excellente avant de partir.",
        },
        {
          "id": "SP15",
          "title": "Le Petit Message",
          "desc":
              "Envoyez un texte 'Comment vas-tu ?' à quelqu'un avec qui vous n'avez pas parlé depuis un mois.",
        },
        {
          "id": "SP16",
          "title": "La Question Ouverte",
          "desc":
              "Demandez à quelqu'un 'Quel est votre endroit préféré à visiter dans cette ville ?'",
        },
        {
          "id": "SP17",
          "title": "Le Plus Petit Risque",
          "desc":
              "Demandez à un étranger s'il sait où se trouvent les toilettes les plus proches.",
        },
        {
          "id": "SP18",
          "title": "L'Éloge de l'Objet",
          "desc":
              "Dites à quelqu'un que vous aimez ses chaussures/son sac/son accessoire.",
        },
        {
          "id": "SP19",
          "title": "La Pause Polie",
          "desc":
              "Attendez que quelqu'un ait fini de parler complètement avant de lui répondre.",
        },
        {
          "id": "SP20",
          "title": "Le Salut Amical",
          "desc":
              "Faites un signe de la main et dites 'Au revoir' à quelqu'un avec qui vous venez d'avoir une brève interaction.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "Chercheur d'Opinions",
          "desc":
              "Demandez l'avis de quelqu'un sur un livre, un film or une chanson.",
        },
        {
          "id": "L2",
          "title": "Le Détail",
          "desc":
              "Posez une question de suivi après que quelqu'un vous ait parlé de lui-même.",
        },
        {
          "id": "L3",
          "title": "La Recommandation",
          "desc":
              "Demandez à un étranger une recommandation pour un bon endroit où manger à proximité.",
        },
        {
          "id": "L4",
          "title": "La Connexion",
          "desc":
              "Trouvez un centre d'intérêt commun avec quelqu'un et parlez-en pendant 2 minutes.",
        },
        {
          "id": "L5",
          "title": "La Main Secourable",
          "desc":
              "Proposez d'aider quelqu'un pour une petite tâche (comme porter un sac).",
        },
        {
          "id": "L6",
          "title": "L'Observation Sociale",
          "desc":
              "Engagez la conversation en vous basant sur quelque chose qui se passe autour de vous.",
        },
        {
          "id": "L7",
          "title": "La Question Ouverte",
          "desc":
              "Demandez à quelqu'un 'Comment vous êtes-vous lancé dans ce métier ?'",
        },
        {
          "id": "L8",
          "title": "L'Écoute Active",
          "desc":
              "Écoutez quelqu'un pendant 3 minutes sans l'interrompre, puis résumez ce qu'il a dit.",
        },
        {
          "id": "L9",
          "title": "Le Rire Partagé",
          "desc":
              "Racontez une courte histoire drôle or une blague à un petit groupe.",
        },
        {
          "id": "L10",
          "title": "La Curiosité",
          "desc":
              "Demandez à quelqu'un d'où il vient et ce qu'il aime dans cet endroit.",
        },
        {
          "id": "L11",
          "title": "L'Intérêt Sincère",
          "desc":
              "Interrogez un collègue sur ses loisirs en dehors du travail.",
        },
        {
          "id": "L12",
          "title": "Le Conseil Doux",
          "desc":
              "Donnez un conseil utile à quelqu'un sur un sujet où vous êtes doué.",
        },
        {
          "id": "L13",
          "title": "L'Accord Groupal",
          "desc":
              "Approuvez le point de vue de quelqu'un lors d'une discussion de groupe.",
        },
        {
          "id": "L14",
          "title": "L'Invitation Casual",
          "desc":
              "Demandez à quelqu'un 'Voudriez-vous nous accompagner pour déjeuner ?'",
        },
        {
          "id": "L15",
          "title": "L'Honnête Réflexion",
          "desc":
              "Dites à quelqu'un 'J'ai vraiment apprécié quand tu as fait X' et expliquez pourquoi.",
        },
        {
          "id": "L16",
          "title": "La Brèche de Curiosité",
          "desc":
              "Demandez à quelqu'un 'Je me suis toujours demandé, comment X fonctionne-t-il vraiment ?'",
        },
        {
          "id": "L17",
          "title": "Liderazgo de Grupo Pequeño",
          "desc":
              "Posez une question qui nécessite que 2 ou 3 personnes d'un groupe répondent.",
        },
        {
          "id": "L18",
          "title": "Le Compliment Genuin",
          "desc":
              "Faites un compliment à quelqu'un sur un trait de sa personnalité (ex: 'Vous êtes un excellent auditeur').",
        },
        {
          "id": "L19",
          "title": "L'Expérience Partagée",
          "desc":
              "Dites 'J'ai déjà été dans cette situation aussi' lors d'une conversation.",
        },
        {
          "id": "L20",
          "title": "La Pause Significative",
          "desc":
              "Laissez un silence s'installer dans une conversation sans se précipiter pour le combler.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "Le Début Courageux",
          "desc":
              "Engagez la conversation avec quelqu'un que vous connaissez peu.",
        },
        {
          "id": "ST2",
          "title": "Le Partage Honnête",
          "desc":
              "Partagez une petite histoire personnelle ou une opinion dans un cadre groupal.",
        },
        {
          "id": "ST3",
          "title": "Le Débat",
          "desc":
              "Exprimez poliment votre désaccord avec l'opinion de quelqu'un et expliquez pourquoi.",
        },
        {
          "id": "ST4",
          "title": "L'Entrée dans le Groupe",
          "desc":
              "Joignez-vous à une conversation de groupe et apportez une phrase réfléchie.",
        },
        {
          "id": "ST5",
          "title": "Le Leader du Sujet",
          "desc":
              "Lancez un nouveau sujet de conversation dans un groupe social.",
        },
        {
          "id": "ST6",
          "title": "La Question Publique",
          "desc":
              "Posez une question lors d'une réunion publique ou dans une salle de classe.",
        },
        {
          "id": "ST7",
          "title": "La Demande Audacieuse",
          "desc":
              "Demandez à un étranger si vous pouvez vous asseoir à côté de lui dans un café ou un parc.",
        },
        {
          "id": "ST8",
          "title": "Le Pont de Conversation",
          "desc":
              "Présentez deux personnes qui ne se connaissent pas et trouvez un point commun.",
        },
        {
          "id": "ST9",
          "title": "Le Besoin Assertif",
          "desc":
              "Demandez poliment à quelqu'un de se déplacer ou d'arrêter de faire quelque chose qui vous gêne.",
        },
        {
          "id": "ST10",
          "title": "Le Conteur",
          "desc":
              "Prenez l'initiative de raconter une histoire à un groupe de 3 personnes ou plus.",
        },
        {
          "id": "ST11",
          "title": "Le Défi Ouvert",
          "desc":
              "Remettez en question une opinion commune dans un groupe de manière amicale et respectueuse.",
        },
        {
          "id": "ST12",
          "title": "L'Initiative Sociale",
          "desc":
              "Soyez la première personne à dire 'Bonjour' à tout le monde en entrant dans une pièce.",
        },
        {
          "id": "ST13",
          "title": "L'Écoute Empathique",
          "desc":
              "Écoutez quelqu'un se confier et apportez une réponse compréhensive.",
        },
        {
          "id": "ST14",
          "title": "La Présentation Publique",
          "desc":
              "Parlez pendant 1 à 2 minutes d'un sujet que vous aimez lors d'un rassemblement social.",
        },
        {
          "id": "ST15",
          "title": "Le Partage Vulnérable",
          "desc":
              "Admettez devant un groupe que vous étiez nerveux pour quelque chose, et riez-en.",
        },
        {
          "id": "ST16",
          "title": "La Limite Établie",
          "desc":
              "Refusez poliment une invitation que vous ne souhaitez pas accepter sans trop vous justifier.",
        },
        {
          "id": "ST17",
          "title": "Le Médiateur Actif",
          "desc":
              "Aidez deux personnes à trouver un terrain d'entente lors d'un désaccord.",
        },
        {
          "id": "ST18",
          "title": "Le Compliment Public",
          "desc":
              "Louvez publiquement l'effort ou la réussite de quelqu'un dans un groupe.",
        },
        {
          "id": "ST19",
          "title": "L'Approche Directe",
          "desc":
              "Demandez directement à quelqu'un une faveur or un conseil dont vous avez besoin.",
        },
        {
          "id": "ST20",
          "title": "Le Pivot de Conversation",
          "desc":
              "Faites glisser doucement une conversation d'un sujet ennuyeux vers un sujet intéressant.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "Le Cadeau",
          "desc":
              "Offrez une petite friandise à quelqu'un en disant 'J'ai pensé que cela vous plairait'.",
        },
        {
          "id": "B2",
          "title": "Le Lead Audacieux",
          "desc":
              "Suggérez un plan or un lieu à visiter à un petit groupe de personnes.",
        },
        {
          "id": "B3",
          "title": "Le Appréciation",
          "desc":
              "Dites à quelqu'un spécifiquement pourquoi vous appréciez sa présence dans votre vie.",
        },
        {
          "id": "B4",
          "title": "Le Hôte Social",
          "desc":
              "Organisez un petit rassemblement or un rendez-vous café pour quelques personnes.",
        },
        {
          "id": "B5",
          "title": "Le Immersion Profonde",
          "desc":
              "Ayez une conversation profonde et significative avec quelqu'un pendant plus de 15 minutes.",
        },
        {
          "id": "B6",
          "title": "Le Pic de Confiance",
          "desc":
              "Engagez la conversation avec quelqu'un que vous trouvez intimidant.",
        },
        {
          "id": "B7",
          "title": "Le Toast Public",
          "desc":
              "Faites un toast court et positif or un hommage à quelqu'un dans un groupe.",
        },
        {
          "id": "B8",
          "title": "Le Établisseur de Limites",
          "desc":
              "Dites 'Non' à une demande avec fermeté mais gentillesse, sans trop vous justifier.",
        },
        {
          "id": "B9",
          "title": "La Demande Directe",
          "desc":
              "Demandez à quelqu'un que vous admirez un entretien de 10 minutes or un mentorat.",
        },
        {
          "id": "B10",
          "title": "Le Lead Émotionnel",
          "desc":
              "Engagez une conversation sur les sentiments or la santé mentale avec un ami.",
        },
        {
          "id": "B11",
          "title": "Le Médiateur Social",
          "desc":
              "Aidez deux personnes à résoudre un petit conflit via une conversation calme.",
        },
        {
          "id": "B12",
          "title": "Le Compliment Audacieux",
          "desc":
              "Dites à un parfait étranger quelque chose que vous admirez sincèrement chez lui.",
        },
        {
          "id": "B13",
          "title": "Le Mouvement de Networking",
          "desc":
              "Présentez-vous à un professionnel de votre domaine et demandez-lui conseil.",
        },
        {
          "id": "B14",
          "title": "La Vérité Courageuse",
          "desc":
              "Dites à quelqu'un une vérité difficile à dire, mais utile pour la relation.",
        },
        {
          "id": "B15",
          "title": "Le Épanouissement Total",
          "desc":
              "Organisez un petit événement social et assurez-vous que chaque invité se sente accueilli.",
        },
        {
          "id": "B16",
          "title": "Le Orateur Public",
          "desc":
              "Proposez-vous pour parler or diriger une petite partie d'une réunion or d'un événement.",
        },
        {
          "id": "B17",
          "title": "Le Lead Vulnérable",
          "desc":
              "Partagez une lutte que vous avez surmontée pour encourager quelqu'un d'autre.",
        },
        {
          "id": "B18",
          "title": "Le Excuse Audacieuse",
          "desc":
              "Engagez une conversation pour vous excuser d'une erreur passée, même si c'était il y a longtemps.",
        },
        {
          "id": "B19",
          "title": "Le Mentor",
          "desc":
              "Offrez votre aide à quelqu'un qui a moins d'expérience que vous dans une compétence.",
        },
        {
          "id": "B20",
          "title": "Le Architecte Social",
          "desc":
              "Créez une nouvelle tradition sociale or une réunion récurrente pour un groupe d'amis.",
        },
      ],
    },
    'hi': {
      "Seedling": [
        {
          "id": "S1",
          "title": "पहला कदम",
          "desc": "आज किसी एक व्यक्ति से नज़रें मिलाएं और मुस्कुराएं।",
        },
        {
          "id": "S2",
          "title": "एक साधारण नमस्ते",
          "desc": "किसी पड़ोसी को 'सुप्रभात' या 'नमस्ते' कहें।",
        },
        {
          "id": "S3",
          "title": "धन्यवाद",
          "desc": "दुकानदार को स्पष्ट रूप से 'धन्यवाद' कहें।",
        },
        {
          "id": "S4",
          "title": "अवलोकन",
          "desc": "किसी अजनबी में कुछ सकारात्मक देखें और मुस्कुराएं।",
        },
        {
          "id": "S5",
          "title": "शांत इशारा",
          "desc": "दूरी से किसी ऐसे व्यक्ति को हाथ हिलाएं जिसे आप पहचानते हैं।",
        },
        {
          "id": "S6",
          "title": "दरवाजा पकड़ना",
          "desc": "अपने पीछे आने वाले व्यक्ति के लिए दरवाजा खुला रखें।",
        },
        {
          "id": "S7",
          "title": "सिर हिलाना",
          "desc":
              "पास से गुजरते समय किसी सहकर्मी को मित्रतापूर्वक सिर हिलाकर अभिवादन करें।",
        },
        {
          "id": "S8",
          "title": "दर्पण अभ्यास",
          "desc":
              "1 मिनट के लिए दर्पण में अपनी 'आत्मविश्वासी मुस्कान' का अभ्यास करें।",
        },
        {
          "id": "S9",
          "title": "संक्षिप्त नज़र",
          "desc":
              "किसी को 2 सेकंड के लिए देखें, फिर मुस्कुराएं और नज़र हटा लें।",
        },
        {
          "id": "S10",
          "title": "शांत प्रशंसा",
          "desc": "किसी के सोशल मीडिया पोस्ट पर एक अच्छा कमेंट लिखें।",
        },
        {
          "id": "S11",
          "title": "स्थान साझा करना",
          "desc":
              "सार्वजनिक क्षेत्र में किसी के बगल में बैठें और तुरंत नज़र न हटाएं।",
        },
        {
          "id": "S12",
          "title": "साधारण स्वीकृति",
          "desc":
              "गलियारे में किसी के पास से गुजरते समय विनम्रता से 'क्षमा करें' कहें।",
        },
        {
          "id": "S13",
          "title": "गर्मजोशी भरा अभिवादन",
          "desc": "किसी डिलीवरी ड्राइवर या कूरियर को 'नमस्ते' कहें।",
        },
        {
          "id": "S14",
          "title": "छोटी सी लहर",
          "desc":
              "किसी बच्चे या पालतू जानवर (मालिक की अनुमति से) को हाथ हिलाएं।",
        },
        {
          "id": "S15",
          "title": "कोमल मुस्कान",
          "desc": "आज तीन अलग-अलग लोगों को देखकर मुस्कुराएं।",
        },
        {
          "id": "S16",
          "title": "आई-कॉन्टैक्ट चुनौती",
          "desc":
              "कैशियर के साथ तब तक नज़रें मिलाएं जब तक कि वह पहले नज़र न हटा ले।",
        },
        {
          "id": "S17",
          "title": "कोमल सांस",
          "desc": "आज किसी सामाजिक स्थान पर जाने से पहले 3 गहरी सांसें लें।",
        },
        {
          "id": "S18",
          "title": "उपस्थिति",
          "desc":
              "भीड़भाड़ वाले क्षेत्र में 5 मिनट तक बिना फोन देखे खड़े रहें।",
        },
        {
          "id": "S19",
          "title": "अनौपचारिक सिर हिलाना",
          "desc":
              "किसी अजनबी को सिर हिलाकर अभिवादन करें जो आपसे नज़रें मिलाता है।",
        },
        {
          "id": "S20",
          "title": "कोमल आवाज़",
          "desc": "दुकान से बाहर निकलते समय किसी को 'आपका दिन शुभ हो' कहें।",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "तारीफ",
          "desc": "किसी सहकर्मी या सहपाठी की सच्ची तारीफ करें।",
        },
        {
          "id": "SP2",
          "title": "प्रश्न",
          "desc": "किसी अजनबी से समय या दिशा-निर्देश पूछें।",
        },
        {
          "id": "SP3",
          "title": "छोटी बातचीत",
          "desc": "किसी से पूछें 'आपका दिन कैसा जा रहा है?' और उत्तर सुनें।",
        },
        {
          "id": "SP4",
          "title": "अनुरोध",
          "desc":
              "किसी विशिष्ट वस्तु को खोजने में मदद के लिए स्टोर कर्मचारी से पूछें।",
        },
        {
          "id": "SP5",
          "title": "ऑर्डर",
          "desc": "एक पेय या भोजन ऑर्डर करें और स्टाफ से पूछें कि वे कैसे हैं।",
        },
        {
          "id": "SP6",
          "title": "अभिवादन",
          "desc": "अपने क्षेत्र में किसी नए व्यक्ति से अपना परिचय दें।",
        },
        {
          "id": "SP7",
          "title": "मौसम की बात",
          "desc":
              "लाइन में प्रतीक्षा करते समय किसी से मौसम के बारे में बात करें।",
        },
        {
          "id": "SP8",
          "title": "साधारण पूछताछ",
          "desc": "किसी सहकर्मी से पूछें 'आपने वीकेंड पर क्या किया?'",
        },
        {
          "id": "SP9",
          "title": "मदद की पेशकश",
          "desc":
              "किसी से पूछें 'क्या आपको इसमें मदद चाहिए?' यदि वे संघर्ष करते दिखें।",
        },
        {
          "id": "SP10",
          "title": "राय",
          "desc":
              "किसी मित्र से किसी छोटी वस्तु के बारे में पूछें 'आप इसके बारे में क्या सोचते हैं?'",
        },
        {
          "id": "SP11",
          "title": "पुष्टि",
          "desc":
              "किसी अजनबी से विवरण की पुष्टि करें (जैसे, 'क्या यह सही लाइन है?').",
        },
        {
          "id": "SP12",
          "title": "साझा स्थान",
          "desc":
              "वातावरण के बारे में एक छोटी टिप्पणी करें (जैसे, 'आज बहुत भीड़ है').",
        },
        {
          "id": "SP13",
          "title": "छोटी मदद",
          "desc": "किसी से टेबल पर कुछ पास करने के लिए कहें (जैसे नैपकिन).",
        },
        {
          "id": "SP14",
          "title": "गर्म प्रतिक्रिया",
          "desc": "जाने से पहले वेटर को बताएं कि खाना बहुत अच्छा था।",
        },
        {
          "id": "SP15",
          "title": "अनौपचारिक हाल-चाल",
          "desc":
              "किसी ऐसे व्यक्ति को 'आप कैसे हैं?' मैसेज भेजें जिससे आपने एक महीने से बात नहीं की है।",
        },
        {
          "id": "SP16",
          "title": "खुला प्रश्न",
          "desc": "किसी से पूछें 'इस शहर में आपकी पसंदीदा जगह कौन सी है?'",
        },
        {
          "id": "SP17",
          "title": "सबसे छोटा जोखिम",
          "desc":
              "किसी अजनबी से पूछें कि क्या वे जानते हैं कि निकटतम शौचालय कहाँ है।",
        },
        {
          "id": "SP18",
          "title": "वस्तु की प्रशंसा",
          "desc": "किसी को बताएं कि आपको उनके जूते/बैग/एक्सेसरी पसंद आई।",
        },
        {
          "id": "SP19",
          "title": "विनम्र ठहराव",
          "desc":
              "किसी के पूरी तरह से बोलना समाप्त करने तक प्रतीक्षा करें और फिर उत्तर दें।",
        },
        {
          "id": "SP20",
          "title": "मैत्रीपूर्ण लहर",
          "desc":
              "किसी ऐसे व्यक्ति को हाथ हिलाकर 'बाय' कहें जिसके साथ आपने अभी संक्षिप्त बातचीत की हो।",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "राय जानना",
          "desc":
              "किसी से किसी किताब, फिल्म या गाने के बारे में उनकी राय पूछें।",
        },
        {
          "id": "L2",
          "title": "विवरण",
          "desc":
              "जब कोई अपने बारे में कुछ बताए, तो उसके बाद एक फॉलो-अप प्रश्न पूछें।",
        },
        {
          "id": "L3",
          "title": "सिफारिश",
          "desc":
              "किसी अजनबी से पास में खाने के लिए एक अच्छी जगह की सिफारिश मांगें।",
        },
        {
          "id": "L4",
          "title": "जुड़ाव",
          "desc":
              "किसी के साथ एक समान रुचि खोजें और उसके बारे में 2 मिनट तक बात करें।",
        },
        {
          "id": "L5",
          "title": "मददगार हाथ",
          "desc": "किसी को छोटे काम (जैसे बैग उठाने) में मदद की पेशकश करें।",
        },
        {
          "id": "L6",
          "title": "सामाजिक अवलोकन",
          "desc": "अपने आसपास हो रही किसी चीज़ के आधार पर बातचीत शुरू करें।",
        },
        {
          "id": "L7",
          "title": "खुला प्रश्न",
          "desc": "किसी से पूछें 'आप इस काम में कैसे आए?'",
        },
        {
          "id": "L8",
          "title": "सक्रिय श्रोता",
          "desc":
              "किसी को 3 मिनट तक बिना टोके सुनें, फिर उन्होंने जो कहा उसका सारांश बताएं।",
        },
        {
          "id": "L9",
          "title": "साझा हंसी",
          "desc": "एक छोटे समूह को एक छोटी, मजेदार कहानी या चुटकुला सुनाएं।",
        },
        {
          "id": "L10",
          "title": "जिज्ञासा",
          "desc":
              "किसी से पूछें कि वे कहाँ से हैं और उन्हें उस जगह के बारे में क्या पसंद है।",
        },
        {
          "id": "L11",
          "title": "सच्ची रुचि",
          "desc": "किसी सहकर्मी से उनके काम के बाहर के शौक के बारे में पूछें।",
        },
        {
          "id": "L12",
          "title": "कोमल सलाह",
          "desc": "किसी को उस चीज़ पर उपयोगी सुझाव दें जिसमें आप अच्छे हैं।",
        },
        {
          "id": "L13",
          "title": "समूह सहमति",
          "desc": "एक छोटे समूह की चर्चा में किसी की बात से सहमत हों।",
        },
        {
          "id": "L14",
          "title": "अनौपचारिक निमंत्रण",
          "desc":
              "किसी से पूछें 'क्या आप हमारे साथ दोपहर के भोजन पर चलना चाहेंगे?'",
        },
        {
          "id": "L15",
          "title": "ईमानदार प्रतिबिंब",
          "desc":
              "किसी को बताएं 'जब आपने X किया तो मैंने वास्तव में उसकी सराहना की' और समझाएं क्यों।",
        },
        {
          "id": "L16",
          "title": "जिज्ञासा अंतर",
          "desc":
              "किसी से पूछें 'मैं हमेशा सोचता था, X वास्तव में कैसे काम करता है?'",
        },
        {
          "id": "L17",
          "title": "छोटा समूह नेतृत्व",
          "desc":
              "एक ऐसा प्रश्न पूछें जिसके उत्तर के लिए समूह के 2 या 3 लोगों की आवश्यकता हो।",
        },
        {
          "id": "L18",
          "title": "सच्ची तारीफ",
          "desc":
              "किसी के व्यक्तित्व गुण की तारीफ करें (जैसे, 'आप एक बहुत अच्छे श्रोता हैं').",
        },
        {
          "id": "L19",
          "title": "साझा अनुभव",
          "desc": "बातचीत के दौरान कहें 'मैं भी उसी स्थिति में रहा हूँ'।",
        },
        {
          "id": "L20",
          "title": "अर्थपूर्ण ठहराव",
          "desc": "बातचीत में सन्नाटे को आने दें, उसे भरने की जल्दबाजी न करें।",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "साहसी शुरुआत",
          "desc":
              "किसी ऐसे व्यक्ति से बातचीत शुरू करें जिसे आप अच्छी तरह नहीं जानते।",
        },
        {
          "id": "ST2",
          "title": "ईमानदार साझाकरण",
          "desc": "समूह सेटिंग में एक छोटी व्यक्तिगत कहानी या राय साझा करें।",
        },
        {
          "id": "ST3",
          "title": "बहस",
          "desc": "विनम्रतापूर्वक किसी की राय से असहमत हों और समझाएं क्यों।",
        },
        {
          "id": "ST4",
          "title": "समूह प्रवेश",
          "desc": "एक समूह बातचीत में शामिल हों और एक विचारशील वाक्य जोड़ें।",
        },
        {
          "id": "ST5",
          "title": "विषय नेतृत्व",
          "desc": "एक सामाजिक समूह में बातचीत का एक नया विषय लाएं।",
        },
        {
          "id": "ST6",
          "title": "सार्वजनिक प्रश्न",
          "desc": "सार्वजनिक बैठक या कक्षा सेटिंग में एक प्रश्न पूछें।",
        },
        {
          "id": "ST7",
          "title": "साहसी अनुरोध",
          "desc":
              "किसी अजनबी से पूछें कि क्या आप कैफे या पार्क में उनके बगल में बैठ सकते हैं।",
        },
        {
          "id": "ST8",
          "title": "बातचीत का सेतु",
          "desc":
              "दो ऐसे लोगों का परिचय कराएं जो एक दूसरे को नहीं जानते और एक समानता खोजें।",
        },
        {
          "id": "ST9",
          "title": "मुखर आवश्यकता",
          "desc":
              "विनम्रतापूर्वक किसी से हटने या कुछ ऐसा करना बंद करने के लिए कहें जो आपको परेशान कर रहा है।",
        },
        {
          "id": "ST10",
          "title": "कहानीकार",
          "desc": "3 या अधिक लोगों के समूह को कहानी सुनाने में नेतृत्व करें।",
        },
        {
          "id": "ST11",
          "title": "खुली चुनौती",
          "desc":
              "समूह में एक आम राय को मित्रवत और सम्मानजनक तरीके से चुनौती दें।",
        },
        {
          "id": "ST12",
          "title": "सामाजिक पहल",
          "desc":
              "कमरे में प्रवेश करते समय हर किसी को 'नमस्ते' कहने वाले पहले व्यक्ति बनें।",
        },
        {
          "id": "ST13",
          "title": "सहानुभूतिपूर्ण श्रवण",
          "desc":
              "किसी को अपनी भड़ास निकालते हुए सुनें और एक सहायक प्रतिक्रिया दें।",
        },
        {
          "id": "ST14",
          "title": "सार्वजनिक प्रस्तुति",
          "desc": "एक सामाजिक सभा में अपने पसंदीदा विषय पर 1-2 मिनट तक बोलें।",
        },
        {
          "id": "ST15",
          "title": "कमजोरी साझा करना",
          "desc":
              "एक समूह में स्वीकार करें कि आप किसी चीज़ को लेकर घबराए हुए थे, और उस पर हंसें।",
        },
        {
          "id": "ST16",
          "title": "सीमा निर्धारित करना",
          "desc":
              "बिना अधिक स्पष्टीकरण दिए, विनम्रतापूर्वक एक निमंत्रण को अस्वीकार करें जिसे आप स्वीकार नहीं करना चाहते।",
        },
        {
          "id": "ST17",
          "title": "सक्रिय मध्यस्थ",
          "desc":
              "किसी असहमति में दो लोगों को बीच का रास्ता खोजने में मदद करें।",
        },
        {
          "id": "ST18",
          "title": "सार्वजनिक प्रशंसा",
          "desc":
              "समूह में किसी के प्रयास या उपलब्धि की सार्वजनिक रूप से प्रशंसा करें।",
        },
        {
          "id": "ST19",
          "title": "सीधा दृष्टिकोण",
          "desc":
              "किसी से सीधे किसी मदद या सलाह के लिए पूछें जिसकी आपको आवश्यकता है।",
        },
        {
          "id": "ST20",
          "title": "बातचीत का मोड़",
          "desc":
              "एक उबाऊ विषय से बातचीत को सहजता से एक दिलचस्प विषय की ओर मोड़ें।",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "उपहार",
          "desc":
              "किसी को एक छोटी सी मिठाई दें और कहें 'मुझे लगा कि आपको यह पसंद आएगा'।",
        },
        {
          "id": "B2",
          "title": "साहसी नेतृत्व",
          "desc":
              "लोगों के एक छोटे समूह को किसी योजना या घूमने की जगह का सुझाव दें।",
        },
        {
          "id": "B3",
          "title": "प्रशंसा",
          "desc":
              "किसी को विशेष रूप से बताएं कि आप उन्हें अपने जीवन में क्यों महत्व देते हैं।",
        },
        {
          "id": "B4",
          "title": "सामाजिक मेजबान",
          "desc":
              "कुछ लोगों के लिए एक छोटा गेट-टुगेदर या कॉफी डेट आयोजित करें।",
        },
        {
          "id": "B5",
          "title": "गहरी बातचीत",
          "desc":
              "किसी के साथ 15 मिनट से अधिक समय तक गहरी, सार्थक बातचीत करें।",
        },
        {
          "id": "B6",
          "title": "आत्मविश्वास का शिखर",
          "desc":
              "किसी ऐसे व्यक्ति के साथ बातचीत शुरू करें जिसे आप डरावना पाते हैं।",
        },
        {
          "id": "B7",
          "title": "सार्वजनिक टोस्ट",
          "desc":
              "समूह में किसी के लिए एक छोटा, सकारात्मक टोस्ट या प्रशंसा करें।",
        },
        {
          "id": "B8",
          "title": "सीमा निर्धारित करना",
          "desc":
              "किसी अनुरोध को दृढ़ता लेकिन विनम्रता से 'ना' कहें, बिना अधिक स्पष्टीकरण के।",
        },
        {
          "id": "B9",
          "title": "सीधा अनुरोध",
          "desc":
              "जिस व्यक्ति की आप प्रशंसा करते हैं, उससे 10 मिनट की बातचीत या मार्गदर्शन मांगें।",
        },
        {
          "id": "B10",
          "title": "भावनात्मक नेतृत्व",
          "desc":
              "किसी मित्र के साथ भावनाओं या मानसिक स्वास्थ्य के बारे में बातचीत शुरू करें।",
        },
        {
          "id": "B11",
          "title": "सामाजिक मध्यस्थ",
          "desc":
              "एक शांत बातचीत के माध्यम से दो लोगों को छोटा विवाद सुलझाने में मदद करें।",
        },
        {
          "id": "B12",
          "title": "साहसी तारीफ",
          "desc":
              "एक पूर्ण अजनबी को बताएं कि आप उनमें वास्तव में किस बात की प्रशंसा करते हैं।",
        },
        {
          "id": "B13",
          "title": "नेटवर्किंग मूव",
          "desc":
              "अपने क्षेत्र के किसी पेशेवर से अपना परिचय कराएं और सलाह मांगें।",
        },
        {
          "id": "B14",
          "title": "साहसी सच्चाई",
          "desc":
              "किसी को ऐसी सच्चाई बताएं जो कहना कठिन हो, लेकिन रिश्ते के लिए उपयोगी हो।",
        },
        {
          "id": "B15",
          "title": "पूर्ण प्रस्फुटन",
          "desc":
              "एक छोटा सामाजिक कार्यक्रम आयोजित करें और सुनिश्चित करें कि हर मेहमान का स्वागत महसूस हो।",
        },
        {
          "id": "B16",
          "title": "सार्वजनिक वक्ता",
          "desc":
              "किसी बैठक या कार्यक्रम के छोटे हिस्से को बोलने या निर्देशित करने के लिए स्वयंसेवा करें।",
        },
        {
          "id": "B17",
          "title": "कमजोरी का नेतृत्व",
          "desc":
              "किसी और को प्रोत्साहित करने के लिए एक संघर्ष साझा करें जिसे आपने पार किया हो।",
        },
        {
          "id": "B18",
          "title": "साहसी क्षमा",
          "desc":
              "किसी पुरानी गलती के लिए माफी मांगने के लिए बातचीत शुरू करें, भले ही वह बहुत समय पहले की हो।",
        },
        {
          "id": "B19",
          "title": "मेंटर",
          "desc":
              "किसी ऐसे व्यक्ति की मदद करें जो किसी कौशल में आपसे कम अनुभवी है।",
        },
        {
          "id": "B20",
          "title": "सामाजिक वास्तुकार",
          "desc":
              "दोस्तों के समूह के लिए एक नई सामाजिक परंपरा या एक आवर्ती मुलाकात बनाएं।",
        },
      ],
    },
    'de': {
      "Seedling": [
        {
          "id": "S1",
          "title": "Der erste Schritt",
          "desc":
              "Suche heute Blickkontakt mit einer Person und lächle sie an.",
        },
        {
          "id": "S2",
          "title": "Ein einfaches Hallo",
          "desc": "Sage einem Nachbarn „Guten Morgen“ oder „Hallo“.",
        },
        {
          "id": "S3",
          "title": "Das Danke",
          "desc": "Sage einem Ladenbesitzer deutlich „Dankeschön“.",
        },
        {
          "id": "S4",
          "title": "Die Beobachtung",
          "desc": "Bemerke etwas Positives an einem Fremden und lächle.",
        },
        {
          "id": "S5",
          "title": "Das leise Winken",
          "desc": "Winke jemandem aus der Ferne zu, den du wiedererkennst.",
        },
        {
          "id": "S6",
          "title": "Tür aufhalten",
          "desc": "Halte jemandem hinter dir die Tür offen.",
        },
        {
          "id": "S7",
          "title": "Das Nicken",
          "desc":
              "Grüße einen Kollegen im Vorbeigehen mit einem freundlichen Nicken.",
        },
        {
          "id": "S8",
          "title": "Der Spiegel",
          "desc": "Übe dein „selbstbewusstes Lächeln“ für 1 Minute im Spiegel.",
        },
        {
          "id": "S9",
          "title": "Der kurze Blick",
          "desc":
              "Schaue jemanden für 2 Sekunden an, lächle dann und blicke weg.",
        },
        {
          "id": "S10",
          "title": "Das stille Lob",
          "desc":
              "Schreibe einen netten Kommentar unter den Social-Media-Beitrag von jemandem.",
        },
        {
          "id": "S11",
          "title": "Den Raum teilen",
          "desc":
              "Setze dich an einem öffentlichen Ort neben jemanden, ohne sofort wegzusehen.",
        },
        {
          "id": "S12",
          "title": "Die einfache Anerkennung",
          "desc":
              "Sage höflich „Entschuldigung“, wenn du an jemandem im Flur vorbeigehst.",
        },
        {
          "id": "S13",
          "title": "Die herzliche Begrüßung",
          "desc": "Sage einem Paketboten oder Lieferanten „Hallo“.",
        },
        {
          "id": "S14",
          "title": "Das kleine Winken",
          "desc":
              "Winke einem Kind oder einem Haustier zu (mit Erlaubnis des Besitzers).",
        },
        {
          "id": "S15",
          "title": "Das sanfte Lächeln",
          "desc": "Lächle heute drei verschiedene Personen an.",
        },
        {
          "id": "S16",
          "title": "Die Blickkontakt-Herausforderung",
          "desc":
              "Halte den Blickkontakt mit einem Kassierer, bis dieser zuerst wegschaut.",
        },
        {
          "id": "S17",
          "title": "Das sanfte Atmen",
          "desc":
              "Atme heute dreimal tief durch, bevor du einen sozialen Raum betrittst.",
        },
        {
          "id": "S18",
          "title": "Die Präsenz",
          "desc":
              "Stehe für 5 Minuten an einem belebten Ort, ohne auf dein Handy zu schauen.",
        },
        {
          "id": "S19",
          "title": "Das beiläufige Nicken",
          "desc": "Nicke einem Fremden zu, der Blickkontakt mit dir sucht.",
        },
        {
          "id": "S20",
          "title": "Die leise Stimme",
          "desc":
              "Sage jemandem beim Verlassen eines Ladens „Einen schönen Tag noch“.",
        },
      ],
      "Sprout": [
        {
          "id": "L1",
          "title": "Meinungssucher",
          "desc":
              "Frage jemanden nach seiner Meinung zu einem Buch, Film oder Lied.",
        },
        {
          "id": "L2",
          "title": "Das Detail",
          "desc":
              "Stelle eine Folgefrage, nachdem dir jemand etwas über sich selbst erzählt hat.",
        },
        {
          "id": "L3",
          "title": "Die Empfehlung",
          "desc":
              "Frage einen Fremden nach einer Empfehlung für ein gutes Restaurant in der Nähe.",
        },
        {
          "id": "L4",
          "title": "Die Gemeinsamkeit",
          "desc":
              "Finde ein gemeinsames Interesse mit jemandem und sprich 2 Minuten lang darüber.",
        },
        {
          "id": "L5",
          "title": "Die helfende Hand",
          "desc":
              "Biete jemandem Hilfe bei einer kleinen Aufgabe an (z. B. beim Tragen einer Tasche).",
        },
        {
          "id": "L6",
          "title": "Die soziale Beobachtung",
          "desc":
              "Beginne ein Gespräch basierend auf etwas, das um euch beide herum passiert.",
        },
        {
          "id": "L7",
          "title": "Die offene Frage",
          "desc":
              "Frage jemanden: „Wie bist du zu dieser Art von Arbeit gekommen?“",
        },
        {
          "id": "L8",
          "title": "Der aktive Zuhörer",
          "desc":
              "Höre jemandem 3 Minuten lang zu, ohne ihn zu unterbrechen, und fasse dann zusammen, was er gesagt hat.",
        },
        {
          "id": "L9",
          "title": "Das gemeinsame Lachen",
          "desc":
              "Erzähle einer kleinen Gruppe eine kurze, lustige Geschichte oder einen Witz.",
        },
        {
          "id": "L10",
          "title": "Die Neugier",
          "desc":
              "Frage jemanden, woher er kommt und was ihm an diesem Ort gefällt.",
        },
        {
          "id": "L11",
          "title": "Das aufrichtige Interesse",
          "desc":
              "Frage einen Kollegen nach seinen Hobbys außerhalb der Arbeit.",
        },
        {
          "id": "L12",
          "title": "Der sanfte Rat",
          "desc":
              "Gib jemandem einen hilfreichen Tipp zu etwas, worin du gut bist.",
        },
        {
          "id": "L13",
          "title": "Das Kopfnicken in der Gruppe",
          "desc":
              "Stimme dem Argument von jemandem in einer kleinen Gruppendiskussion zu.",
        },
        {
          "id": "L14",
          "title": "Die ungezwungene Einladung",
          "desc":
              "Frage jemanden: „Möchtest du mit uns zum Mittagessen kommen?“",
        },
        {
          "id": "L15",
          "title": "Die ehrliche Rückmeldung",
          "desc":
              "Sage jemandem: „Ich habe es wirklich geschätzt, als du X getan hast“, und erkläre warum.",
        },
        {
          "id": "L16",
          "title": "Die Wissenslücke",
          "desc":
              "Frage jemanden: „Ich habe mich schon immer gefragt, wie funktioniert X eigentlich?“",
        },
        {
          "id": "L17",
          "title": "Die kleine Gruppenführung",
          "desc":
              "Stelle eine Frage, deren Beantwortung von 2 oder 3 Personen in einer Gruppe verlangt wird.",
        },
        {
          "id": "L18",
          "title": "Das aufrichtige Kompliment",
          "desc":
              "Mache jemandem ein Kompliment zu einer Charaktereigenschaft (z. B. „Du bist ein toller Zuhörer“).",
        },
        {
          "id": "L19",
          "title": "Die geteilte Erfahrung",
          "desc":
              "Sage während eines Gesprächs: „In dieser Situation war ich auch schon mal“.",
        },
        {
          "id": "L20",
          "title": "Die vielsagende Pause",
          "desc":
              "Lasse ein Schweigen in einem Gespräch zu, ohne dich zu beeilen, es zu füllen.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "Opinion Seeker",
          "desc": "Ask someone for their opinion on a book, movie, or song.",
        },
        {
          "id": "L2",
          "title": "The Detail",
          "desc":
              "Ask a follow-up question after someone tells you something about themselves.",
        },
        {
          "id": "L3",
          "title": "The Recommendation",
          "desc":
              "Ask a stranger for a recommendation for a good place to eat nearby.",
        },
        {
          "id": "L4",
          "title": "The Connection",
          "desc":
              "Find a common interest with someone and talk about it for 2 minutes.",
        },
        {
          "id": "L5",
          "title": "The Helpful Hand",
          "desc":
              "Offer to help someone with a small task (like carrying a bag).",
        },
        {
          "id": "L6",
          "title": "The Social Observation",
          "desc":
              "Start a conversation based on something happening around you both.",
        },
        {
          "id": "L7",
          "title": "The Open Ended Question",
          "desc": "Ask someone 'How did you get into this line of work?'",
        },
        {
          "id": "L8",
          "title": "The Active Listener",
          "desc":
              "Listen to someone for 3 minutes without interrupting, then summarize what they said.",
        },
        {
          "id": "L9",
          "title": "The Shared Laugh",
          "desc": "Tell a short, funny story or a joke to a small group.",
        },
        {
          "id": "L10",
          "title": "The Curiosity",
          "desc":
              "Ask someone where they are from and what they like about that place.",
        },
        {
          "id": "L11",
          "title": "The Sincere Interest",
          "desc": "Ask a colleague about their hobbies outside of work.",
        },
        {
          "id": "L12",
          "title": "The Soft Advice",
          "desc": "Give someone a helpful tip on something you are good at.",
        },
        {
          "id": "L13",
          "title": "The Group Nod",
          "desc": "Agree with someone's point in a small group discussion.",
        },
        {
          "id": "L14",
          "title": "The Casual Invitation",
          "desc": "Ask someone 'Would you like to join us for lunch?'",
        },
        {
          "id": "L15",
          "title": "The Honest Reflection",
          "desc":
              "Tell someone 'I really appreciated it when you did X' and explain why.",
        },
        {
          "id": "L16",
          "title": "The Curiosity Gap",
          "desc":
              "Ask someone 'I've always wondered, how does X actually work?'",
        },
        {
          "id": "L17",
          "title": "The Small Group Lead",
          "desc":
              "Ask a question that requires 2 or 3 people in a group to answer.",
        },
        {
          "id": "L18",
          "title": "The Genuine Compliment",
          "desc":
              "Compliment someone on a personality trait (e.g., 'You're a great listener').",
        },
        {
          "id": "L19",
          "title": "The Shared Experience",
          "desc":
              "Say 'I've been in that situation too' during a conversation.",
        },
        {
          "id": "L20",
          "title": "The Meaningful Pause",
          "desc":
              "Allow a silence to happen in a conversation without rushing to fill it.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "Der mutige Anfang",
          "desc": "Beginne ein Gespräch mit jemandem, den du nicht gut kennst.",
        },
        {
          "id": "ST2",
          "title": "Das ehrliche Teilen",
          "desc":
              "Teile eine kleine persönliche Geschichte oder Meinung in einer Gruppe.",
        },
        {
          "id": "ST3",
          "title": "Die Debatte",
          "desc":
              "Widersprich der Meinung von jemandem höflich und erkläre warum.",
        },
        {
          "id": "ST4",
          "title": "Der Gruppeneinstieg",
          "desc":
              "Beteilige dich an einem Gespräch in einer Gruppe und trage einen durchdachten Satz bei.",
        },
        {
          "id": "ST5",
          "title": "Die Themenführung",
          "desc": "Bringe ein neues Gesprächsthema in eine soziale Gruppe ein.",
        },
        {
          "id": "ST6",
          "title": "Die öffentliche Frage",
          "desc":
              "Stelle eine Frage in einer öffentlichen Versammlung oder im Unterricht.",
        },
        {
          "id": "ST7",
          "title": "Die mutige Bitte",
          "desc":
              "Frage einen Fremden, ob du dich in einem Café oder Park neben ihn setzen darfst.",
        },
        {
          "id": "ST8",
          "title": "Die Gesprächsbrücke",
          "desc":
              "Stelle zwei Personen einander vor, die sich nicht kennen, und finde eine Gemeinsamkeit.",
        },
        {
          "id": "ST9",
          "title": "Das durchsetzungsstarke Bedürfnis",
          "desc":
              "Bitte jemanden höflich, beiseitezutreten oder mit etwas aufzuhören, das dich stört.",
        },
        {
          "id": "ST10",
          "title": "Der Geschichtenerzähler",
          "desc":
              "Übernimm die Führung beim Erzählen einer Geschichte vor einer Gruppe von 3 oder mehr Personen.",
        },
        {
          "id": "ST11",
          "title": "Die offene Herausforderung",
          "desc":
              "Hinterfrage eine gängige Meinung in einer Gruppe auf eine freundliche und respektvolle Weise.",
        },
        {
          "id": "ST12",
          "title": "Die soziale Initiative",
          "desc":
              "Sei die erste Person, die beim Betreten eines Raumes alle mit „Hallo“ grüßt.",
        },
        {
          "id": "ST13",
          "title": "Das empathische Zuhören",
          "desc":
              "Höre jemandem zu, der seinen Frust ablädt, und gib eine unterstützende Antwort.",
        },
        {
          "id": "ST14",
          "title": "Die öffentliche Präsentation",
          "desc":
              "Sprich bei einem sozialen Treffen 1–2 Minuten lang über ein Thema, das du liebst.",
        },
        {
          "id": "ST15",
          "title": "Das verletzliche Teilen",
          "desc":
              "Gib vor einer Gruppe zu, dass du wegen etwas nervös warst, und lacht gemeinsam darüber.",
        },
        {
          "id": "ST16",
          "title": "Die Grenzsetzung",
          "desc":
              "Lehne eine Einladung, an der du nicht teilnehmen möchtest, höflich ab, ohne dich zu rechtfertigen.",
        },
        {
          "id": "ST17",
          "title": "Der aktive Vermittler",
          "desc":
              "Hilf zwei Personen, bei einer Meinungsverschiedenheit einen Kompromiss zu finden.",
        },
        {
          "id": "ST18",
          "title": "Das öffentliche Kompliment",
          "desc":
              "Lobe den Einsatz oder die Leistung von jemandem öffentlich in einer Gruppe.",
        },
        {
          "id": "ST19",
          "title": "Die direkte Ansprache",
          "desc":
              "Frage jemanden direkt nach einem Gefallen oder einem Rat, den du benötigst.",
        },
        {
          "id": "ST20",
          "title": "Der Gesprächswechsel",
          "desc":
              "Leite ein Gespräch reibungslos von einem langweiligen Thema zu einem interessanten über.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "Das Geschenk",
          "desc":
              "Gib jemandem eine kleine Aufmerksamkeit und sage: „Ich dachte, das gefällt dir“.",
        },
        {
          "id": "B2",
          "title": "Die mutige Führung",
          "desc":
              "Schlage einer kleinen Gruppe von Menschen einen Plan oder einen Ort vor, den man besuchen könnte.",
        },
        {
          "id": "B3",
          "title": "Die Wertschätzung",
          "desc":
              "Sage jemandem ganz konkret, warum du es schätzt, ihn in deinem Leben zu haben.",
        },
        {
          "id": "B4",
          "title": "Der soziale Gastgeber",
          "desc":
              "Organisiere ein kleines Treffen oder eine Verabredung zum Kaffeetrinken für ein paar Leute.",
        },
        {
          "id": "B5",
          "title": "Das tiefe Gespräch",
          "desc":
              "Führe ein tiefes, bedeutungsvolles Gespräch mit jemandem für mehr als 15 Minuten.",
        },
        {
          "id": "B6",
          "title": "Der Gipfel des Selbstvertrauens",
          "desc":
              "Beginne ein Gespräch mit jemandem, den du einschüchternd findest.",
        },
        {
          "id": "B7",
          "title": "Der öffentliche Toast",
          "desc":
              "Sprich einen kurzen, positiven Toast oder ein Lob auf jemanden in einer Gruppe aus.",
        },
        {
          "id": "B8",
          "title": "Der Grenzsetzer",
          "desc":
              "Lehne eine Bitte fest, aber freundlich ab, ohne dich dafür zu rechtfertigen.",
        },
        {
          "id": "B9",
          "title": "Die direkte Anfrage",
          "desc":
              "Frage jemanden, den du bewunderst, nach einem 10-minütigen Gespräch oder Mentoring.",
        },
        {
          "id": "B10",
          "title": "Die emotionale Führung",
          "desc":
              "Beginne mit einem Freund ein Gespräch über Gefühle oder mentale Gesundheit.",
        },
        {
          "id": "B11",
          "title": "Der soziale Vermittler",
          "desc":
              "Hilf zwei Menschen, einen kleinen Konflikt durch ein ruhiges Gespräch zu lösen.",
        },
        {
          "id": "B12",
          "title": "Das mutige Kompliment",
          "desc":
              "Sage einem völlig Fremden etwas, das du aufrichtig an ihm bewunderst.",
        },
        {
          "id": "B13",
          "title": "Der Netzwerkschritt",
          "desc":
              "Stelle dich einer Fachkraft aus deinem Berufsfeld vor und bitte sie um Rat.",
        },
        {
          "id": "B14",
          "title": "Die mutige Wahrheit",
          "desc":
              "Sage jemandem eine Wahrheit, die zwar schwierig, aber hilfreich für die Beziehung ist.",
        },
        {
          "id": "B15",
          "title": "Die volle Blüte",
          "desc":
              "Veranstalte ein kleines soziales Event und sorge dafür, dass sich jeder Gast willkommen fühlt.",
        },
        {
          "id": "B16",
          "title": "Der öffentliche Redner",
          "desc":
              "Melde dich freiwillig, um bei einem Treffen oder einer Veranstaltung zu sprechen oder einen kleinen Teil zu leiten.",
        },
        {
          "id": "B17",
          "title": "Die verletzliche Führung",
          "desc":
              "Teile eine persönliche Herausforderung, die du überwunden hast, um jemand anderen zu ermutigen.",
        },
        {
          "id": "B18",
          "title": "Die mutige Entschuldigung",
          "desc":
              "Suche das Gespräch, um dich für einen vergangenen Fehler zu entschuldigen, selbst wenn dieser lange zurückliegt.",
        },
        {
          "id": "B19",
          "title": "Der Mentor",
          "desc":
              "Biete jemandem, der weniger erfahren ist als du, Hilfe bei einer Fähigkeit an.",
        },
        {
          "id": "B20",
          "title": "Der soziale Architekt",
          "desc":
              "Ergänze eine neue soziale Tradition oder ein wiederkehrendes Treffen für eine Gruppe von Freunden.",
        },
      ],
    },
    'ur': {
      "Seedling": [
        {
          "id": "S1",
          "title": "پہلا قدم",
          "desc": "آج کسی ایک شخص سے نظریں ملائیں اور مسکرائیں۔",
        },
        {
          "id": "S2",
          "title": "ایک سادہ ہیلو",
          "desc": "کسی پڑوسی کو 'گڈ مارننگ' یا 'ہیلو' کہیں۔",
        },
        {
          "id": "S3",
          "title": "شکریہ ادا کرنا",
          "desc": "کسی دکاندار کو واضح طور پر 'شکریہ' کہیں۔",
        },
        {
          "id": "S4",
          "title": "مشاہدہ کرنا",
          "desc": "کسی اجنبی میں کوئی مثبت چیز نوٹ کریں اور مسکرائیں۔",
        },
        {
          "id": "S5",
          "title": "خاموشی سے ہاتھ ہلانا",
          "desc": "دور سے کسی ایسے شخص کو ہاتھ ہلائیں جسے آپ پہچانتے ہوں۔",
        },
        {
          "id": "S6",
          "title": "دروازہ کھلا رکھنا",
          "desc": "اپنے پیچھے آنے والے کسی شخص کے لیے دروازہ کھلا رکھیں۔",
        },
        {
          "id": "S7",
          "title": "سر ہلانا",
          "desc":
              "کسی ساتھی کے پاس سے گزرتے ہوئے اسے دوستانہ انداز میں سر ہلا کر سلام کریں۔",
        },
        {
          "id": "S8",
          "title": "آئینہ",
          "desc":
              "آئینے کے سامنے کھڑے ہو کر 1 منٹ تک اپنی 'پر اعتماد مسکراہٹ' کی مشق کریں۔",
        },
        {
          "id": "S9",
          "title": "مختصر نظر",
          "desc":
              "کسی کو 2 سیکنڈ کے لیے دیکھیں، پھر مسکرائیں اور نظریں ہٹا لیں۔",
        },
        {
          "id": "S10",
          "title": "خاموشی سے تعریف",
          "desc": "کسی کی سوشل میڈیا پوسٹ پر ایک اچھا تبصرہ لکھیں۔",
        },
        {
          "id": "S11",
          "title": "جگہ شیئر کرنا",
          "desc":
              "کسی عوامی جگہ پر کسی کے پاس بیٹھ جائیں اور فوراً نظریں نہ ہٹائیں۔",
        },
        {
          "id": "S12",
          "title": "سادہ اعتراف",
          "desc":
              "راہداری میں کسی کے پاس سے گزرتے وقت شائستگی سے 'ایکسکیوز می' کہیں۔",
        },
        {
          "id": "S13",
          "title": "گرم جوش استقبال",
          "desc": "کسی ڈیلیوری ڈرائیور یا کورئیر کو 'ہائے' کہیں۔",
        },
        {
          "id": "S14",
          "title": "چھوٹا سا اشارہ",
          "desc": "کسی بچے یا پالتو جانور کو ہاتھ ہلائیں (مالک کی اجازت سے)۔",
        },
        {
          "id": "S15",
          "title": "نرم مسکراہٹ",
          "desc": "آج تین الگ الگ لوگوں کو دیکھ کر مسکرائیں۔",
        },
        {
          "id": "S16",
          "title": "آئی کانٹیکٹ چیلنج",
          "desc":
              "کیشیئر سے اس وقت تک نظریں ملائے رکھیں جب تک وہ خود پہلے نظر نہ ہٹا لے۔",
        },
        {
          "id": "S17",
          "title": "آرام سے سانس لینا",
          "desc": "آج کسی بھی سماجی جگہ میں داخل ہونے سے پہلے 3 گہرے سانس لیں۔",
        },
        {
          "id": "S18",
          "title": "موجودگی",
          "desc":
              "اپنے فون کو دیکھے بغیر 5 منٹ تک کسی پرہجوم جگہ پر کھڑے رہیں۔",
        },
        {
          "id": "S19",
          "title": "سر کا اشارہ",
          "desc": "کسی ایسے اجنبی کو دیکھ کر سر ہلائیں جو آپ سے نظریں ملائے۔",
        },
        {
          "id": "S20",
          "title": "نرم آواز",
          "desc": "کسی دکان سے نکلتے وقت کسی کو 'آپ کا دن اچھا گزرے' کہیں۔",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "تعریف کرنا",
          "desc": "اپنے کسی ساتھی یا ہم جماعت کی سچی اور دلی تعریف کریں۔",
        },
        {
          "id": "SP2",
          "title": "سوال پوچھنا",
          "desc": "کسی اجنبی سے وقت یا راستے کے بارے میں پوچھیں۔",
        },
        {
          "id": "SP3",
          "title": "مختصر گفتگو",
          "desc":
              "کسی سے پوچھیں 'آپ کا دن کیسا گزر رہا ہے؟' اور اس کا جواب غور سے سنیں۔",
        },
        {
          "id": "SP4",
          "title": "درخواست کرنا",
          "desc":
              "دکان کے کسی ملازم سے کسی مخصوص چیز کو ڈھونڈنے میں مدد مانگیں۔",
        },
        {
          "id": "SP5",
          "title": "آرڈر دینا",
          "desc":
              "کسی کھانے یا پینے کی چیز کا آرڈر دیں اور عملے سے پوچھیں کہ وہ کیسے ہیں۔",
        },
        {
          "id": "SP6",
          "title": "سلام دعا",
          "desc": "اپنے علاقے میں کسی نئے شخص سے اپنا تعارف کروائیں۔",
        },
        {
          "id": "SP7",
          "title": "موسم کی بات چیت",
          "desc":
              "لائن میں کھڑے ہو کر انتظار کے دوران کسی سے موسم کا تذکرہ کریں۔",
        },
        {
          "id": "SP8",
          "title": "سادہ پوچھ گچھ",
          "desc":
              "اپنے ساتھ کام کرنے والے سے پوچھیں 'آپ نے ویک اینڈ پر کیا کیا؟'",
        },
        {
          "id": "SP9",
          "title": "مدد کی پیشکش",
          "desc":
              "اگر کوئی پریشان یا مشکل میں نظر آئے تو اس سے پوچھیں 'کیا آپ کو اس میں مدد کی ضرورت ہے؟'۔",
        },
        {
          "id": "SP10",
          "title": "رائے مانگنا",
          "desc":
              "کسی دوست کو کوئی چھوٹی چیز دکھا کر پوچھیں 'آپ کا اس کے بارے میں کیا خیال ہے؟'۔",
        },
        {
          "id": "SP11",
          "title": "تصدیق کرنا",
          "desc":
              "کسی اجنبی سے کسی بات کی تصدیق کریں (جیسے کہ، 'کیا یہ صحیح لائن ہے؟')۔",
        },
        {
          "id": "SP12",
          "title": "مشترکہ جگہ",
          "desc":
              "ماحول کے بارے میں ایک چھوٹا سا تبصرہ کریں (جیسے کہ، 'یہاں واقعی بہت رش ہے')۔",
        },
        {
          "id": "SP13",
          "title": "چھوٹا سا کام",
          "desc":
              "ٹیبل پر بیٹھے ہوئے کسی سے کوئی چیز (جیسے نیپکن) اپنی طرف بڑھانے کا کہیں۔",
        },
        {
          "id": "SP14",
          "title": "اچھا فیڈ بیک",
          "desc": "جانے سے پہلے ویٹر کو بتائیں کہ کھانا بہت لاجواب تھا۔",
        },
        {
          "id": "SP15",
          "title": "خیریت معلوم کرنا",
          "desc":
              "کسی ایسے شخص کو 'آپ کیسے ہیں؟' کا میسج بھیجیں جس سے آپ نے ایک مہینے سے بات نہ کی ہو۔",
        },
        {
          "id": "SP16",
          "title": "کھلا سوال",
          "desc":
              "کسی سے پوچھیں 'اس شہر میں آپ کی سب سے پسندیدہ گھومنے والی جگہ کون سی ہے؟'۔",
        },
        {
          "id": "SP17",
          "title": "سب سے چھوٹا رسک",
          "desc":
              "کسی اجنبی سے پوچھیں کہ کیا وہ جانتے ہیں کہ سب سے قریبی واش روم کہاں ہے۔",
        },
        {
          "id": "SP18",
          "title": "چیزوں کی تعریف",
          "desc": "کسی کو بتائیں کہ آپ کو ان کے جوتے/بیگ/ایکسیسری پسند آئی ہے۔",
        },
        {
          "id": "SP19",
          "title": "شائستہ وقفہ",
          "desc":
              "کسی کو جواب دینے سے پہلے اس کی بات مکمل طور پر ختم ہونے کا انتظار کریں۔",
        },
        {
          "id": "SP20",
          "title": "دوستانہ الوداع",
          "desc":
              "کسی ایسے شخص کو ہاتھ ہلائیں اور 'بائے' کہیں جس کے ساتھ ابھی آپ کی مختصر سی بات چیت ہوئی ہو۔",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "رائے مانگنے والا",
          "desc": "کسی سے کسی کتاب، فلم یا گانے پر ان کی رائے پوچھیں۔",
        },
        {
          "id": "L2",
          "title": "تفصیل",
          "desc":
              "جب کوئی آپ کو اپنے بارے میں کچھ بتائے تو اس سے متعلق ایک مزید سوال پوچھیں۔",
        },
        {
          "id": "L3",
          "title": "سفارش",
          "desc": "کسی اجنبی سے قریب ہی کھانے کی کسی اچھی جگہ کی سفارش پوچھیں۔",
        },
        {
          "id": "L4",
          "title": "مشترکہ تعلق",
          "desc":
              "کسی کے ساتھ کوئی مشترکہ دلچسپی تلاش کریں اور اس پر 2 منٹ تک بات کریں۔",
        },
        {
          "id": "L5",
          "title": "مددگار ہاتھ",
          "desc":
              "کسی چھوٹے کام میں مدد کی پیشکش کریں (جیسے کہ سامان کا تھیلا اٹھانا)۔",
        },
        {
          "id": "L6",
          "title": "سماجی مشاہدہ",
          "desc":
              "آپ دونوں کے ارد گرد ہونے والے کسی واقعے کو بنیاد بنا کر گفتگو کا آغاز کریں۔",
        },
        {
          "id": "L7",
          "title": "کھلا سوال",
          "desc": "کسی سے پوچھیں 'آپ اس پیشے یا کام میں کیسے آئے؟'۔",
        },
        {
          "id": "L8",
          "title": "سرگرم سامع",
          "desc":
              "کسی کی بات بغیر مداخلت کے 3 منٹ تک سنیں، پھر انہوں نے جو کہا اس کا خلاصہ کریں۔",
        },
        {
          "id": "L9",
          "title": "مشترکہ ہنسی",
          "desc": "کسی چھوٹے گروپ کو ایک مختصر، مزاحیہ کہانی یا لطیفہ سنائیں۔",
        },
        {
          "id": "L10",
          "title": "تجسس",
          "desc":
              "کسی سے پوچھیں کہ ان کا تعلق کہاں سے ہے اور انہیں اس جگہ کے بارے میں کیا پسند ہے۔",
        },
        {
          "id": "L11",
          "title": "سچی دلچسپی",
          "desc": "کسی ساتھی سے کام کے علاوہ ان کے مشاغل کے بارے میں پوچھیں۔",
        },
        {
          "id": "L12",
          "title": "نرم مشورہ",
          "desc":
              "جس چیز میں آپ ماہر ہوں، اس کے بارے میں کسی کو ایک مفید معلوماتی ٹپ دیں۔",
        },
        {
          "id": "L13",
          "title": "گروپ میں تائید",
          "desc":
              "کسی چھوٹے گروپ کی گفتگو میں کسی کی بات سے سر ہلا کر اتفاق کریں۔",
        },
        {
          "id": "L14",
          "title": "غیر رسمی دعوت",
          "desc":
              "کسی سے پوچھیں 'کیا آپ لنچ میں ہمارے ساتھ شامل ہونا چاہیں گے؟'۔",
        },
        {
          "id": "L15",
          "title": "سچا اظہار",
          "desc":
              "کسی کو بتائیں 'جب آپ نے ایکس (X) کیا تو مجھے واقعی بہت اچھا لگا تھا' اور اس کی وجہ بتائیں۔",
        },
        {
          "id": "L16",
          "title": "تجسس کا فاصلہ",
          "desc":
              "کسی سے پوچھیں 'میں ہمیشہ سوچتا ہوں کہ ایکس (X) اصل میں کیسے کام کرتا ہے؟'۔",
        },
        {
          "id": "L17",
          "title": "گروپ گفتگو کی شروعات",
          "desc":
              "ایسا سوال پوچھیں جس کا جواب دینے کے لیے گروپ میں 2 یا 3 لوگوں کی ضرورت ہو۔",
        },
        {
          "id": "L18",
          "title": "سچی تعریف",
          "desc":
              "کسی کی شخصیت کی خوبی پر ان کی تعریف کریں (جیسے کہ، 'آپ ایک بہت اچھے سننے والے ہیں')۔",
        },
        {
          "id": "L19",
          "title": "مشترکہ تجربہ",
          "desc": "گفتگو کے دوران کہیں 'میں بھی اس صورتحال سے گزر چکا ہوں'۔",
        },
        {
          "id": "L20",
          "title": "بامعنی وقفہ",
          "desc":
              "گفتگو میں خاموشی کے لمحے کو برقرار رہنے دیں اور اسے فوراً ختم کرنے کی جلدی نہ کریں۔",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "بہادرانہ شروعات",
          "desc":
              "کسی ایسے شخص کے ساتھ گفتگو شروع کریں جسے آپ اچھی طرح نہیں جانتے۔",
        },
        {
          "id": "ST2",
          "title": "سچا اشتراک",
          "desc": "کسی گروپ میں اپنی کوئی چھوٹی ذاتی کہانی یا رائے شیئر کریں۔",
        },
        {
          "id": "ST3",
          "title": "بحث و تکرار",
          "desc": "کسی کی رائے سے شائستگی سے اختلاف کریں اور اس کی وجہ بتائیں۔",
        },
        {
          "id": "ST4",
          "title": "گروپ میں شمولیت",
          "desc":
              "کسی گروپ کی گفتگو میں شامل ہوں اور ایک سوچا سمجھا جملہ اس میں پیش کریں۔",
        },
        {
          "id": "ST5",
          "title": "موضوع کی رہنمائی",
          "desc": "کسی سماجی گروپ میں گفتگو کے لیے ایک نیا موضوع پیش کریں۔",
        },
        {
          "id": "ST6",
          "title": "عوامی سوال",
          "desc": "کسی عوامی میٹنگ یا کلاس روم کے ماحول میں سوال پوچھیں۔",
        },
        {
          "id": "ST7",
          "title": "جرأت مندانہ درخواست",
          "desc":
              "کسی اجنبی سے پوچھیں کہ کیا آپ کسی کیفے یا پارک میں ان کے ساتھ بیٹھ سکتے ہیں۔",
        },
        {
          "id": "ST8",
          "title": "گفتگو کا پل",
          "desc":
              "دو ایسے لوگوں کا ایک دوسرے سے تعارف کروائیں جو ایک دوسرے کو نہیں جانتے اور ان میں کوئی مشترکہ بات ڈھونڈیں۔",
        },
        {
          "id": "ST9",
          "title": "واضح ضرورت",
          "desc":
              "کسی سے شائستگی سے کہیں کہ وہ ہٹ جائیں یا کوئی ایسا کام بند کر دیں جو آپ کو پریشان کر رہا ہو۔",
        },
        {
          "id": "ST10",
          "title": "قصہ گو",
          "desc":
              "3 یا اس سے زیادہ لوگوں کے گروپ کو کوئی کہانی سنانے میں پہل کریں۔",
        },
        {
          "id": "ST11",
          "title": "کھلا چیلنج",
          "desc":
              "کسی گروپ میں ایک عام رائے کو دوستانہ اور احترام کے دائرے میں رہ کر چیلنج کریں۔",
        },
        {
          "id": "ST12",
          "title": "سماجی پہل",
          "desc":
              "کسی کمرے میں داخل ہوتے وقت سب کو سب سے پہلے 'ہیلو' کہنے والے شخص بنیں۔",
        },
        {
          "id": "ST13",
          "title": "ہمدردانہ سماعت",
          "desc":
              "کسی کی پریشانی یا غصے کی باتیں غور سے سنیں اور اس کا ایک معاون جواب دیں۔",
        },
        {
          "id": "ST14",
          "title": "عوامی پیشکش",
          "desc":
              "کسی سماجی اجتماع میں اپنے پسندیدہ موضوع پر 1 سے 2 منٹ تک گفتگو کریں۔",
        },
        {
          "id": "ST15",
          "title": "کمزوری کا اعتراف",
          "desc":
              "کسی گروپ کے سامنے اعتراف کریں کہ آپ کسی چیز کے بارے میں گھبرا رہے تھے، اور مل کر اس پر ہنسیں۔",
        },
        {
          "id": "ST16",
          "title": "حدود کا تعین",
          "desc":
              "کسی ایسی دعوت کو جہاں آپ جانا نہیں چاہتے، بغیر کسی لمبی وضاحت کے شائستگی سے معذرت کر لیں۔",
        },
        {
          "id": "ST17",
          "title": "سرگرم ثالث",
          "desc":
              "کسی اختلاف یا بحث میں دو لوگوں کو کسی درمیانی راستے پر لانے میں مدد کریں۔",
        },
        {
          "id": "ST18",
          "title": "عوامی تعریف",
          "desc":
              "کسی گروپ میں علانیہ طور پر کسی کی کوشش یا کامیابی کی تعریف کریں۔",
        },
        {
          "id": "ST19",
          "title": "براہ راست نقطہ نظر",
          "desc":
              "کسی سے براہ راست کسی مدد یا مشورے کی درخواست کریں جس کی آپ کو ضرورت ہو۔",
        },
        {
          "id": "ST20",
          "title": "گفتگو کا رخ موڑنا",
          "desc":
              "کسی گفتگو کو بورنگ موضوع سے کسی دلچسپ موضوع کی طرف آسانی سے منتقل کریں۔",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "تحفہ",
          "desc":
              "کسی کو کوئی چھوٹی سی چیز یا کھانے پینے کی چیز دیں اور کہیں 'میں نے سوچا آپ کو یہ پسند آئے گا'۔",
        },
        {
          "id": "B2",
          "title": "جرأت مندانہ قیادت",
          "desc":
              "لوگوں کے ایک چھوٹے سے گروپ کو کسی جگہ جانے یا کسی منصوبے کی تجویز دیں۔",
        },
        {
          "id": "B3",
          "title": "قدردانی",
          "desc":
              "کسی کو خاص طور پر بتائیں کہ آپ ان کا اپنی زندگی میں ہونا کیوں پسند اور قدر کرتے ہیں۔",
        },
        {
          "id": "B4",
          "title": "سماجی میزبان",
          "desc":
              "کچھ لوگوں کے لیے ایک چھوٹی سی ملاقات یا کافی ڈیٹ کا اہتمام کریں۔",
        },
        {
          "id": "B5",
          "title": "گہری گفتگو",
          "desc":
              "کسی کے ساتھ 15 منٹ سے زیادہ دیر تک گہری اور معنی خیز گفتگو کریں۔",
        },
        {
          "id": "B6",
          "title": "اعتماد کی چوٹی",
          "desc":
              "کسی ایسے شخص کے ساتھ بات چیت شروع کریں جس سے آپ مرعوب یا خوفزدہ ہوتے ہوں۔",
        },
        {
          "id": "B7",
          "title": "علانیہ تحسین",
          "desc":
              "کسی گروپ میں کسی شخص کے لیے ایک مختصر، مثبت تعریفی کلمات کہیں یا ان کی کوشش کو سراہیں۔",
        },
        {
          "id": "B8",
          "title": "حدود کا تعین کرنے والا",
          "desc":
              "بغیر کسی لمبی وضاحت کے، کسی فرمائش یا درخواست پر مضبوطی مگر نرمی سے 'ناں' کہیں۔",
        },
        {
          "id": "B9",
          "title": "براہ راست درخواست",
          "desc":
              "کسی ایسے شخص سے جس کی آپ تعریف کرتے ہیں، 10 منٹ کی بات چیت یا رہنمائی کی درخواست کریں۔",
        },
        {
          "id": "B10",
          "title": "جذباتی پہل",
          "desc":
              "اپنے کسی دوست کے ساتھ جذبات یا ذہنی صحت کے موضوع پر بات چیت کا آغاز کریں۔",
        },
        {
          "id": "B11",
          "title": "سماجی ثالث",
          "desc":
              "پرسکون گفتگو کے ذریعے دو لوگوں کے درمیان ایک چھوٹے سے تنازع کو حل کرنے میں مدد کریں۔",
        },
        {
          "id": "B12",
          "title": "بہادرانہ تعریف",
          "desc":
              "کسی بالکل اجنبی شخص کو کوئی ایسی بات بتائیں جو آپ واقعی ان میں پسند یا رشک کرتے ہوں۔",
        },
        {
          "id": "B13",
          "title": "نیٹ ورکنگ قدم",
          "desc":
              "اپنے شعبے کے کسی ماہر یا پیشہ ور شخص سے اپنا تعارف کروائیں اور ان سے مشورہ مانگیں۔",
        },
        {
          "id": "B14",
          "title": "حوصلہ مندانہ سچائی",
          "desc":
              "کسی کو کوئی ایسی سچائی بتائیں جو کہنی مشکل ہو لیکن رشتے یا تعلق کے لیے فائدہ مند ہو۔",
        },
        {
          "id": "B15",
          "title": "کامل کھلنا",
          "desc":
              "ایک چھوٹی سی سماجی تقریب کی ميزبانی کریں اور اس بات کو یقینی بنائیں کہ ہر مہمان خوش آئند محسوس کرے۔",
        },
        {
          "id": "B16",
          "title": "عوامی مقرر",
          "desc":
              "کسی میٹنگ یا تقریب کے ایک چھوٹے سے حصے کی صدارت کرنے یا بولنے کے لیے خود رضاکارانہ پیش کریں۔",
        },
        {
          "id": "B17",
          "title": "کمزوری کے ساتھ رہنمائی",
          "desc":
              "کسی دوسرے کی حوصلہ افزائی کے لیے اپنی کسی ایسی مشکل یا آزمائش کو شیئر کریں جس پر آپ نے قابو پا لیا ہو۔",
        },
        {
          "id": "B18",
          "title": "بہادرانہ معذرت",
          "desc":
              "ماضی کی کسی غلطی پر معافی مانگنے کے لیے گفتگو کی پہل کریں، چاہے وہ کتنی ہی پرانی بات کیوں نہ ہو۔",
        },
        {
          "id": "B19",
          "title": "رہنما (مینٹر)",
          "desc":
              "کسی ایسے شخص کو کسی مہارت یا ہنر میں مدد کی پیشکش کریں جو آپ سے کم تجربہ کار ہو۔",
        },
        {
          "id": "B20",
          "title": "سماجی معمار",
          "desc":
              "دوستوں کے ایک گروپ کے لیے ایک نئی سماجی روایت یا بار بار ہونے والی ملاقات (میٹ اپ) کا آغاز کریں۔",
        },
      ],
    },
    'ar': {
      "Seedling": [
        {
          "id": "S1",
          "title": "الخطوة الأولى",
          "desc": "تواصل بالعين وابسم لشخص واحد اليوم.",
        },
        {
          "id": "S2",
          "title": "تحية بسيطة",
          "desc": "قل 'صباح الخير' أو 'مرحباً' لأحد الجيران.",
        },
        {
          "id": "S3",
          "title": "كلمة شكر",
          "desc": "قل 'شكراً لك' بوضوح لصاحب المتجر.",
        },
        {
          "id": "S4",
          "title": "الملاحظة",
          "desc": "لاحظ شيئاً إيجابياً في شخص غريب وابتسم.",
        },
        {
          "id": "S5",
          "title": "تلويحة هادئة",
          "desc": "لوّح بيدك لشخص تعرفه من مسافة بعيدة.",
        },
        {
          "id": "S6",
          "title": "إمساك الباب",
          "desc": "أمسك الباب مفتوحاً لشخص يسير خلفك.",
        },
        {
          "id": "S7",
          "title": "إيماءة الرأس",
          "desc": "أومئ برأسك بشكل ودي لزميل أثناء مرورك بجانبه.",
        },
        {
          "id": "S8",
          "title": "المرآة",
          "desc": "تدرب على 'ابتسامتك الواثقة' أمام المرآة لمدة دقيقة واحدة.",
        },
        {
          "id": "S9",
          "title": "نظرة خاطفة",
          "desc": "انظر إلى شخص ما لمدة ثانيتين، ثم ابتسم وانظر بعيداً.",
        },
        {
          "id": "S10",
          "title": "الثناء الصامت",
          "desc":
              "اكتب تعليقاً لطيفاً على منشور شخص ما على وسائل التواصل الاجتماعي.",
        },
        {
          "id": "S11",
          "title": "مشاركة المكان",
          "desc":
              "اجلس بجانب شخص ما في منطقة عامة دون أن تنظر بعيداً على الفور.",
        },
        {
          "id": "S12",
          "title": "الاعتذار البسيط",
          "desc": "قل 'معذرة' بأدب عند المرور بجانب شخص ما في الممر.",
        },
        {
          "id": "S13",
          "title": "التحية الدافئة",
          "desc": "قل 'مرحباً' لسائق التوصيل أو عامل البريد.",
        },
        {
          "id": "S14",
          "title": "تلويحة صغيرة",
          "desc": "لوّح بيدك لطفل أو حيوان أليف (بإذن صاحبه).",
        },
        {
          "id": "S15",
          "title": "الابتسامة الناعمة",
          "desc": "ابتسم لثلاثة أشخاص مختلفين اليوم.",
        },
        {
          "id": "S16",
          "title": "تحدي التواصل البصري",
          "desc":
              "حافظ على التواصل البصري مع الصراف (الكاشير) حتى ينظر بعيداً أولاً.",
        },
        {
          "id": "S17",
          "title": "التنفس الهادئ",
          "desc": "خذ 3 أنفاس عميقة قبل دخول أي مساحة اجتماعية اليوم.",
        },
        {
          "id": "S18",
          "title": "الحضور",
          "desc": "قف في منطقة مزدحمة لمدة 5 دقائق دون النظر إلى هاتفك.",
        },
        {
          "id": "S19",
          "title": "الإيماءة العفوية",
          "desc": "أومئ برأسك لشخص غريب يتواصل معك بصرياً.",
        },
        {
          "id": "S20",
          "title": "الصوت الناعم",
          "desc": "قل 'أتمنى لك يوماً سعيداً' لشخص ما أثناء مغادرتك المتجر.",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "الإطراء",
          "desc": "وجّه إطراءً صادقاً لزميل في العمل أو في الدراسة.",
        },
        {
          "id": "SP2",
          "title": "السؤال",
          "desc": "اسأل شخصاً غريباً عن الوقت أو الاتجاهات.",
        },
        {
          "id": "SP3",
          "title": "حديث عابر",
          "desc": "اسأل شخصاً ما 'كيف يسير يومك؟' واستمع إلى الإجابة.",
        },
        {
          "id": "SP4",
          "title": "الطلب",
          "desc": "اسأل أحد موظفي المتجر للمساعدة في العثور على عنصر معين.",
        },
        {
          "id": "SP5",
          "title": "الطلب بالمطعم",
          "desc": "اطلب مشروباً أو طعاماً واسأل الموظفين عن أحوالهم.",
        },
        {
          "id": "SP6",
          "title": "التحية والتعارف",
          "desc": "عرّف نفسك لشخص جديد في منطقتك.",
        },
        {
          "id": "SP7",
          "title": "حديث الطقس",
          "desc": "اذكر حالة الطقس لشخص ما أثناء الانتظار في الطابور.",
        },
        {
          "id": "SP8",
          "title": "الاستفسار البسيط",
          "desc": "اسأل زميلاً في العمل 'ماذا فعلت في عطلة نهاية الأسبوع؟'",
        },
        {
          "id": "SP9",
          "title": "عرض المساعدة",
          "desc":
              "اسأل شخصاً ما 'هل تحتاج إلى مساعدة في ذلك؟' إذا بدا أنه يعاني.",
        },
        {
          "id": "SP10",
          "title": "الرأي",
          "desc": "اسأل صديقاً 'ما رأيك في هذا؟' بخصوص غرض صغير.",
        },
        {
          "id": "SP11",
          "title": "التأكيد",
          "desc":
              "تأكد من تفصيل ما مع شخص غريب (مثل: 'هل هذا هو الطابور الصحيح؟').",
        },
        {
          "id": "SP12",
          "title": "المساحة المشتركة",
          "desc":
              "قم بإبداء تعليق صغير حول البيئة المحيطة (مثل: 'المكان مزدحم حقاً').",
        },
        {
          "id": "SP13",
          "title": "المعروف الصغير",
          "desc": "اسأل شخصاً ما أن يمرر لك شيئاً (مثل منديل) على الطاولة.",
        },
        {
          "id": "SP14",
          "title": "الانطباع الدافئ",
          "desc": "أخبر النادل أن الطعام كان رائعاً قبل المغادرة.",
        },
        {
          "id": "SP15",
          "title": "الاطمئنان العابر",
          "desc":
              "أرسل رسالة نصية تسأل فيها 'كيف حالك؟' لشخص لم تتحدث معه منذ شهر.",
        },
        {
          "id": "SP16",
          "title": "السؤال المفتوح",
          "desc": "اسأل شخصاً ما 'ما هو مكانك المفضل للزيارة في هذه المدينة؟'",
        },
        {
          "id": "SP17",
          "title": "أقل مخاطرة",
          "desc": "اسأل شخصاً غريباً إذا كان يعرف أين توجد أقرب دورة مياه.",
        },
        {
          "id": "SP18",
          "title": "المدح بغرض ما",
          "desc": "أخبر شخصاً ما أنك معجب بحذائه/حقيبته/إكسسواراته.",
        },
        {
          "id": "SP19",
          "title": "التوقف المؤدب",
          "desc": "انتظر حتى ينتهي شخص ما من التحدث تماماً قبل الرد عليه.",
        },
        {
          "id": "SP20",
          "title": "التلويحة الودية",
          "desc": "لوّح بيدك وقل 'وداعاً' لشخص حظيت معه للتو بتفاعل قصير.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "الباحث عن الآراء",
          "desc": "اسأل شخصاً ما عن رأيه في كتاب، أو فيلم، أو أغنية.",
        },
        {
          "id": "L2",
          "title": "التفصيل",
          "desc":
              "اطرح سؤالاً استباقياً للمتابعة بعد أن يخبرك شخص ما بشيء عن نفسه.",
        },
        {
          "id": "L3",
          "title": "التوصية",
          "desc":
              "اسأل شخصاً غريباً عن توصية بمكان جيد لتناول الطعام في مكان قريب.",
        },
        {
          "id": "L4",
          "title": "الرابط المشترك",
          "desc": "ابحث عن اهتمام مشترك مع شخص ما وتحدث عنه لمدة دقيقتين.",
        },
        {
          "id": "L5",
          "title": "اليد المساعدة",
          "desc": "اعرض مساعدة شخص ما في مهمة صغيرة (مثل حمل حقيبة).",
        },
        {
          "id": "L6",
          "title": "الملاحظة الاجتماعية",
          "desc": "ابدأ محادثة بناءً على شيء يحدث حولكما معاً.",
        },
        {
          "id": "L7",
          "title": "السؤال المفتوح",
          "desc": "اسأل شخصاً ما 'كيف دخلت في هذا المجال من العمل؟'",
        },
        {
          "id": "L8",
          "title": "المستمع الفعّال",
          "desc": "استمع إلى شخص ما لمدة 3 دقائق دون مقاطعة، ثم لخص ما قاله.",
        },
        {
          "id": "L9",
          "title": "الضحكة المشتركة",
          "desc": "أخبر قصة قصيرة مضحكة أو نكتة لمجموعة صغيرة من الناس.",
        },
        {
          "id": "L10",
          "title": "التجسس المعرفي",
          "desc": "اسأل شخصاً ما من أين هو وماذا يعجبه في ذلك المكان.",
        },
        {
          "id": "L11",
          "title": "الاهتمام الصادق",
          "desc": "اسأل زميلاً لك عن هواياته خارج نطاق العمل.",
        },
        {
          "id": "L12",
          "title": "النصيحة اللطيفة",
          "desc": "قدم لشخص ما نصيحة مفيدة حول شيء تجيده أنت.",
        },
        {
          "id": "L13",
          "title": "إيماءة المجموعة",
          "desc": "وافق على وجهة نظر شخص ما في مناقشة مجموعة صغيرة.",
        },
        {
          "id": "L14",
          "title": "الدعوة العفوية",
          "desc": "اسأل شخصاً ما 'هل ترغب في الانضمام إلينا لتناول الغداء؟'",
        },
        {
          "id": "L15",
          "title": "الانعكاس الصادق",
          "desc":
              "أخبر شخصاً ما 'لقد قدرت حقاً عندما فعلت كذا (X)' واشرح له السبب.",
        },
        {
          "id": "L16",
          "title": "فجوة الفضول",
          "desc":
              "اسأل شخصاً ما 'لقد تساءلت دائماً، كيف يعمل كذا (X) في الواقع؟'",
        },
        {
          "id": "L17",
          "title": "قيادة المجموعة الصغيرة",
          "desc": "اطرح سؤالاً يتطلب إجابة من شخصين أو ثلاثة أشخاص في مجموعة.",
        },
        {
          "id": "L18",
          "title": "الإطراء الحقيقي",
          "desc": "امدح شخصاً ما على سمة شخصية فيه (مثل: 'أنت مستمع رائع').",
        },
        {
          "id": "L19",
          "title": "التجربة المشتركة",
          "desc": "قل 'لقد مررت بهذا الموقف أيضاً' أثناء محادثة ما.",
        },
        {
          "id": "L20",
          "title": "التوقف ذو المغزى",
          "desc": "اسمح بحدوث لحظة صمت في محادثة ما دون التسرع في ملئها.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "البداية الشجاعة",
          "desc": "ابدأ محادثة مع شخص لا تعرفه جيداً.",
        },
        {
          "id": "ST2",
          "title": "المشاركة الصادقة",
          "desc": "شارك قصة شخصية صغيرة أو رأياً في إطار مجموعة.",
        },
        {
          "id": "ST3",
          "title": "النقاش",
          "desc": "خالف رأي شخص ما بأدب واشرح له السبب.",
        },
        {
          "id": "ST4",
          "title": "الدخول في المجموعة",
          "desc": "انضم إلى محادثة جماعية وشارك فيها بجملة مدروسة.",
        },
        {
          "id": "ST5",
          "title": "قيادة الموضوع",
          "desc": "اطرح موضوعاً جديداً للمحادثة في مجموعة اجتماعية.",
        },
        {
          "id": "ST6",
          "title": "السؤال العلني",
          "desc": "اطرح سؤالاً في اجتماع عام أو في بيئة الفصل الدراسي.",
        },
        {
          "id": "ST7",
          "title": "الطلب الجريء",
          "desc":
              "اسأل شخصاً غريباً إذا كان بإمكانك الجلوس بجانبه في مقهى أو حديقة.",
        },
        {
          "id": "ST8",
          "title": "جسر المحادثة",
          "desc":
              "عرّف شخصين لا يعرفان بعضهما البعض وابحث عن قاسم مشترك بينهما.",
        },
        {
          "id": "ST9",
          "title": "الحاجة الحازمة",
          "desc": "اطلب من شخص ما بأدب أن يتحرك أو يتوقف عن فعل شيء يزعجك.",
        },
        {
          "id": "ST10",
          "title": "الحكواتي",
          "desc":
              "خذ زمام المبادرة في رواية قصة لمجموعة مكونة من 3 أشخاص أو أكثر.",
        },
        {
          "id": "ST11",
          "title": "التحدي المفتوح",
          "desc": "تحدّ رأياً شائعاً في مجموعة بطريقة ودية ومحترمة.",
        },
        {
          "id": "ST12",
          "title": "المبادرة الاجتماعية",
          "desc": "كن أول شخص يقول 'مرحباً' للجميع عند دخول الغرفة.",
        },
        {
          "id": "ST13",
          "title": "الاستماع التعاطفي",
          "desc": "استمع لشخص يفرغ شحنته الغاضبة وقدم له استجابة داعمة.",
        },
        {
          "id": "ST14",
          "title": "التقديم العلني",
          "desc": "تحدث لمدة 1-2 دقيقة عن موضوع تحبه في تجمع اجتماعي.",
        },
        {
          "id": "ST15",
          "title": "المشاركة الضعيفة",
          "desc":
              "اعترف لمجموعة أنك كنت متوتراً بشأن شيء ما، واضحكوا معاً على ذلك.",
        },
        {
          "id": "ST16",
          "title": "وضع الحدود",
          "desc":
              "ارفض بأدب دعوة لا ترغب في حضورها دون الإفراط في الشرح والاعتذار.",
        },
        {
          "id": "ST17",
          "title": "الوسيط الفعّال",
          "desc": "ساعد شخصين في العثور على أرضية مشتركة في خلاف ما.",
        },
        {
          "id": "ST18",
          "title": "الإطراء العلني",
          "desc": "امدح جهد شخص ما أو إنجازه علناً أمام مجموعة.",
        },
        {
          "id": "ST19",
          "title": "النهج المباشر",
          "desc": "اسأل شخصاً ما مباشرة عن معروف أو نصيحة تحتاج إليها.",
        },
        {
          "id": "ST20",
          "title": "محور المحادثة",
          "desc":
              "انقل محادثة ما بسلاسة من موضوع ممل إلى موضوع آخر مثير للاهتمام.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "الهدية",
          "desc":
              "قدّم حلوى أو هدية صغيرة لشخص ما وقل له 'اعتقدت أن هذا سيعجبك'.",
        },
        {
          "id": "B2",
          "title": "القيادة الجريئة",
          "desc": "اقترح خطة أو مكاناً لزيارته على مجموعة صغيرة من الناس.",
        },
        {
          "id": "B3",
          "title": "التقدير",
          "desc": "أخبر شخصاً ما بشكل محدد عن سبب تقديرك لوجوده في حياتك.",
        },
        {
          "id": "B4",
          "title": "المضيف الاجتماعي",
          "desc":
              "نظّم تجمعاً صغيراً أو موعداً لتناول القهوة لعدد قليل من الناس.",
        },
        {
          "id": "B5",
          "title": "الغوص العميق",
          "desc": "خض محادثة عميقة وذات مغزى مع شخص ما لأكثر من 15 دقيقة.",
        },
        {
          "id": "B6",
          "title": "قمة الثقة",
          "desc": "ابدأ محادثة مع شخص تجده مهيباً أو مخيفاً بالنسبة لك.",
        },
        {
          "id": "B7",
          "title": "التحية العلنية",
          "desc": "قم بإلقاء نخب قصير إيجابي أو ثناء علني لشخص ما في مجموعة.",
        },
        {
          "id": "B8",
          "title": "واضع الحدود",
          "desc":
              "قل 'لا' لطلب ما بحزم ولكن بلطف، دون الإفراط في الشرح والتبرير.",
        },
        {
          "id": "B9",
          "title": "الطلب المباشر",
          "desc":
              "اسأل شخصاً تعجب به لإجراء محادثة لمدة 10 دقائق أو لطلب الإرشاد منه.",
        },
        {
          "id": "B10",
          "title": "القيادة العاطفية",
          "desc": "ابدأ محادثة حول المشاعر أو الصحة النفسية مع صديق لك.",
        },
        {
          "id": "B11",
          "title": "الوسيط الاجتماعي",
          "desc": "ساعد شخصين في حل نزاع صغير من خلال محادثة هادئة.",
        },
        {
          "id": "B12",
          "title": "الإطراء الجريء",
          "desc": "أخبر شخصاً غريباً تماماً عن شيء تعجب به فيه بصدق.",
        },
        {
          "id": "B13",
          "title": "خطوة بناء العلاقات",
          "desc": "عرّف نفسك لشخص مهني متخصص في مجالك واطلب منه النصيحة.",
        },
        {
          "id": "B14",
          "title": "الحقيقة الشجاعة",
          "desc":
              "أخبر شخصاً ما بحقيقة قد تكون صعبة ولكنها مفيدة لاستمرار العلاقة.",
        },
        {
          "id": "B15",
          "title": "الازدهار الكامل",
          "desc":
              "استضف حدثاً اجتماعياً صغيراً واحرص على أن يشعر كل ضيف بالترحيب.",
        },
        {
          "id": "B16",
          "title": "المتحدث العلني",
          "desc": "تطوع للتحدث أو قيادة جزء صغير من اجتماع أو حدث ما.",
        },
        {
          "id": "B17",
          "title": "القيادة العفوية الضعيفة",
          "desc": "شارك تحدياً أو صراعاً تغلبت عليه لتشجيع شخص آخر.",
        },
        {
          "id": "B18",
          "title": "الاعتذار الجريء",
          "desc":
              "ابدأ محادثة للاعتذار عن خطأ ما في الماضي، حتى لو كان ذلك منذ زمن طويل.",
        },
        {
          "id": "B19",
          "title": "الموجّه",
          "desc": "اعرض مساعدة شخص أقل خبرة منك لتطوير مهارة معينة.",
        },
        {
          "id": "B20",
          "title": "المهندس الاجتماعي",
          "desc":
              "ابتكر تقليداً اجتماعياً جديداً أو لقاءً دورياً متكرراً لمجموعة من الأصدقاء.",
        },
      ],
    },
    'ja': {
      "Seedling": [
        {"id": "S1", "title": "最初の一歩", "desc": "今日、誰か一人の人と目を合わせて微笑んでみましょう。"},
        {
          "id": "S2",
          "title": "シンプルな挨拶",
          "desc": "近所の人に「おはようございます」または「こんにちは」と言ってみましょう。",
        },
        {
          "id": "S3",
          "title": "感謝の言葉",
          "desc": "お店の店員さんにハッキリと「ありがとうございます」と伝えてみましょう。",
        },
        {
          "id": "S4",
          "title": "ポジティブな観察",
          "desc": "見知らぬ人の素敵なところを見つけて、心の中で微笑んでみましょう。",
        },
        {"id": "S5", "title": "静かな手振り", "desc": "遠くに見える知り合いの人に、そっと手を振ってみましょう。"},
        {
          "id": "S6",
          "title": "ドアを開けて待つ",
          "desc": "後ろから来る人のために、ドアを開けて待ってあげましょう。",
        },
        {"id": "S7", "title": "軽い会釈", "desc": "すれ違う同僚に、親しみを込めて軽く会釈をしてみましょう。"},
        {
          "id": "S8",
          "title": "鏡の前の練習",
          "desc": "鏡の前で1分間、あなたの「自信に満ちた笑顔」を練習してみましょう。",
        },
        {
          "id": "S9",
          "title": "短い視線",
          "desc": "誰かと2秒間目を合わせ、微笑んでから視線をそらしてみましょう。",
        },
        {
          "id": "S10",
          "title": "静かな称賛",
          "desc": "誰かのSNSの投稿に、親切で素敵なコメントを書き込んでみましょう。",
        },
        {
          "id": "S11",
          "title": "空間の共有",
          "desc": "公共の場所で誰かの隣に座り、すぐに視線をそらさずに過ごしてみましょう。",
        },
        {
          "id": "S12",
          "title": "丁寧な一言",
          "desc": "廊下で人とすれ違うときに、丁寧に「失礼します」と言ってみましょう。",
        },
        {
          "id": "S13",
          "title": "温かい挨拶",
          "desc": "配達員や郵便屋さんに「お疲れ様です」や「こんにちは」と声をかけてみましょう。",
        },
        {
          "id": "S14",
          "title": "小さな手振り",
          "desc": "子どもやペットに（飼い主の許可を得て）手を振ってみましょう。",
        },
        {
          "id": "S15",
          "title": "優しい笑顔",
          "desc": "今日、3人の異なる人に向けて優しい笑顔を浮かべてみましょう。",
        },
        {
          "id": "S16",
          "title": "アイコンタクト挑戦",
          "desc": "レジの店員さんが先に目をそらすまで、アイコンタクトを維持してみましょう。",
        },
        {
          "id": "S17",
          "title": "穏やかな呼吸",
          "desc": "今日、社交的な場所に入る前に深呼吸を3回してみましょう。",
        },
        {
          "id": "S18",
          "title": "スマホを見ない時間",
          "desc": "混雑した場所に5分間立ち、スマホを一度も見ずに過ごしてみましょう。",
        },
        {
          "id": "S19",
          "title": "自然なうなずき",
          "desc": "目が合った見知らぬ人に対して、軽くうなずいてみましょう。",
        },
        {
          "id": "S20",
          "title": "優しい声かけ",
          "desc": "お店を出るときに、店員さんに「ありがとうございました」と言ってみましょう。",
        },
      ],
      "Sprout": [
        {"id": "SP1", "title": "褒め言葉", "desc": "同僚やクラスメイトに、心からの褒め言葉を伝えてみましょう。"},
        {"id": "SP2", "title": "道を聞く", "desc": "見知らぬ人に時間や目的地への道を尋ねてみましょう。"},
        {
          "id": "SP3",
          "title": "世間話",
          "desc": "誰かに「今日のご気分はいかがですか？」と聞き、その答えに耳を傾けてみましょう。",
        },
        {
          "id": "SP4",
          "title": "探し物の相談",
          "desc": "お店のスタッフに、特定のアイテムを探すのを手伝ってもらうよう声をかけてみましょう。",
        },
        {
          "id": "SP5",
          "title": "注文のひと工夫",
          "desc": "飲み物や食べ物を注文する際、スタッフに「お忙しいですか？」など一言声をかけてみましょう。",
        },
        {
          "id": "SP6",
          "title": "新しい出会い",
          "desc": "身近な環境にいる新しい人に、自分の自己紹介をしてみましょう。",
        },
        {
          "id": "SP7",
          "title": "天気の話",
          "desc": "列に並んで待っている間、隣の人に天気についての話を振ってみましょう。",
        },
        {
          "id": "SP8",
          "title": "シンプルな質問",
          "desc": "同僚に「週末は何をして過ごしましたか？」と聞いてみましょう。",
        },
        {
          "id": "SP9",
          "title": "手助けの提案",
          "desc": "困っていそうな人がいたら、「何かお手伝いしましょうか？」と声をかけてみましょう。",
        },
        {
          "id": "SP10",
          "title": "意見を求める",
          "desc": "小さな持ち物について、友人に「これどう思う？」と意見を聞いてみましょう。",
        },
        {
          "id": "SP11",
          "title": "確認の声かけ",
          "desc": "見知らぬ人に詳細を確認してみましょう（例：「この列で合っていますか？」）。",
        },
        {
          "id": "SP12",
          "title": "空間の感想",
          "desc": "周囲の環境について小さな一言をつぶやいてみましょう（例：「今日はすごく混んでいますね」）。",
        },
        {
          "id": "SP13",
          "title": "小さな小さなお願い",
          "desc": "テーブルの席で、誰かに「そこのナプキンを取っていただけますか？」と頼んでみましょう。",
        },
        {
          "id": "SP14",
          "title": "温かいフィードバック",
          "desc": "店を出る前に、ウェイターに「食事がとても美味しかったです」と伝えてみましょう。",
        },
        {
          "id": "SP15",
          "title": "気軽なメッセージ",
          "desc": "ここ1ヶ月ほど話していない人に、「元気にしてる？」とメッセージを送ってみましょう。",
        },
        {
          "id": "SP16",
          "title": "オープンな質問",
          "desc": "誰かに「この街で一番おすすめの場所はどこですか？」と尋ねてみましょう。",
        },
        {
          "id": "SP17",
          "title": "小さなリスク",
          "desc": "見知らぬ人に、一番近いお手洗いがどこにあるか知っているか尋ねてみましょう。",
        },
        {
          "id": "SP18",
          "title": "持ち物を褒める",
          "desc": "誰かの靴、バッグ、またはアクセサリーを「素敵ですね」と褒めてみましょう。",
        },
        {
          "id": "SP19",
          "title": "丁寧な傾聴",
          "desc": "相手が話し終えるのを完全に待ってから、自分の返答を始めてみましょう。",
        },
        {
          "id": "SP20",
          "title": "見送り",
          "desc": "少しだけ言葉を交わした人に、手を振って「さようなら」と言ってみましょう。",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "意見の探求",
          "desc": "本、映画、または音楽について、誰かに感想や意見を求めてみましょう。",
        },
        {
          "id": "L2",
          "title": "一歩踏み込んだ質問",
          "desc": "相手が自分自身について話してくれた後、それに関連する質問をもう一つ重ねてみましょう。",
        },
        {
          "id": "L3",
          "title": "おすすめの質問",
          "desc": "見知らぬ人に、この近くで美味しいご飯が食べられる場所を知っているかおすすめを聞いてみましょう。",
        },
        {
          "id": "L4",
          "title": "共通点の発見",
          "desc": "誰かと共通の趣味や関心事を見つけ、それについて2分間話を広げてみましょう。",
        },
        {
          "id": "L5",
          "title": "差し伸べる手",
          "desc": "小さな荷物を持ってあげるなど、誰かのちょっとした作業を手伝う提案をしてみましょう。",
        },
        {
          "id": "L6",
          "title": "状況からの会話",
          "desc": "二人の周りでその瞬間に起きている出来事をきっかけに、自然に会話を始めてみましょう。",
        },
        {
          "id": "L7",
          "title": "背景を尋ねる",
          "desc": "誰かに「どうしてこのお仕事を始められたのですか？」と尋ねてみましょう。",
        },
        {
          "id": "L8",
          "title": "アクティブ・リスニング",
          "desc": "相手の話を遮らずに3分間じっくり聞き、その後に聞いた内容を簡単に要約して伝えてみましょう。",
        },
        {
          "id": "L9",
          "title": "笑いの共有",
          "desc": "小さなグループに向けて、ちょっとした面白い話やジョークを披露してみましょう。",
        },
        {
          "id": "L10",
          "title": "興味を持つ",
          "desc": "誰かに出身地を尋ね、その場所のどんなところが好きなのか聞いてみましょう。",
        },
        {
          "id": "L11",
          "title": "心からの関心",
          "desc": "同僚に、仕事以外での趣味や休日の過ごし方について尋ねてみましょう。",
        },
        {
          "id": "L12",
          "title": "優しいアドバイス",
          "desc": "自分が得意なことについて、誰かに役立つアドバイスやコツを教えてあげましょう。",
        },
        {
          "id": "L13",
          "title": "グループでの共感",
          "desc": "小さなグループのディスカッションで、誰かの意見にしっかりと同意を示してみましょう。",
        },
        {
          "id": "L14",
          "title": "気軽なお誘い",
          "desc": "誰かに「もしよければ、一緒にランチに行きませんか？」と声をかけてみましょう。",
        },
        {
          "id": "L15",
          "title": "感謝のフィードバック",
          "desc": "誰かに「あなたが〜をしてくれたとき、本当に嬉しかったです」とその具体的な理由を伝えてみましょう。",
        },
        {
          "id": "L16",
          "title": "疑問の解消",
          "desc": "誰かに「ずっと気になっていたのですが、それって実際どういう仕組みなんですか？」と尋ねてみましょう。",
        },
        {
          "id": "L17",
          "title": "会話のパスまわし",
          "desc": "グループ内の2〜3人が答える必要があるような問いかけを投げかけてみましょう。",
        },
        {
          "id": "L18",
          "title": "内面の称賛",
          "desc": "相手の内面や性格的な長所を褒めてみましょう（例：「いつもお話を聞くのがお上手ですね」）。",
        },
        {
          "id": "L19",
          "title": "共感の共有",
          "desc": "会話の途中で「私もまったく同じ状況を経験したことがあります」と伝えてみましょう。",
        },
        {
          "id": "L20",
          "title": "沈黙を受け入れる",
          "desc": "会話の中に生まれる静かな沈黙を、慌てて言葉で埋めようとせずに、ゆったりと受け入れてみましょう。",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "勇敢な始まり",
          "desc": "まだあまりよく知らない人と、自分から進んで会話を始めてみましょう。",
        },
        {
          "id": "ST2",
          "title": "自己開示",
          "desc": "グループの集まりの中で、ちょっとした自分の個人的なエピソードや意見を発言してみましょう。",
        },
        {
          "id": "ST3",
          "title": "意見の交わし合い",
          "desc": "誰かの意見に対して、礼儀正しく「私はこう思います」と異なる視点を説明してみましょう。",
        },
        {
          "id": "ST4",
          "title": "会話への参加",
          "desc": "すでに始まっているグループの会話に入り、意味のある一言を付け足してみましょう。",
        },
        {
          "id": "ST5",
          "title": "テーマの提示",
          "desc": "社交的な集まりの中で、自分から新しい会話のトピックを切り出してみましょう。",
        },
        {
          "id": "ST6",
          "title": "公の場での質問",
          "desc": "全体会議や学校のクラス授業の場で、手を挙げて質問をしてみましょう。",
        },
        {
          "id": "ST7",
          "title": "大胆なお願い",
          "desc": "カフェや公園のベンチで、見知らぬ人に「隣に座ってもいいですか？」と聞いてみましょう。",
        },
        {
          "id": "ST8",
          "title": "架け橋となる存在",
          "desc": "お互いを知らない二人の友人を紹介し、二人の共通点を見つけて会話を繋げてみましょう。",
        },
        {
          "id": "ST9",
          "title": "主張の表現",
          "desc": "少し困っている状況の際、相手に対して「少しだけ移動していただけますか？」などと丁寧に主張してみましょう。",
        },
        {
          "id": "ST10",
          "title": "語り手になる",
          "desc": "3人以上のグループの前で、自分が中心となってひとつのエピソードを最後まで話してみましょう。",
        },
        {
          "id": "ST11",
          "title": "友好的な問題提起",
          "desc": "グループ内の一般的な通説に対して、友好的かつ敬意を払った方法で疑問を投げかけてみましょう。",
        },
        {
          "id": "ST12",
          "title": "主導権の挨拶",
          "desc": "部屋に入るとき、部屋にいる全員に向けて自分から一番最初に「こんにちは！」と声をかけてみましょう。",
        },
        {
          "id": "ST13",
          "title": "感情を受け止める",
          "desc": "誰かの愚痴や悩みをじっくりと聞き、相手に寄り添ったサポートの言葉を返してあげましょう。",
        },
        {
          "id": "ST14",
          "title": "ミニスピーチ",
          "desc": "社交的な集まりの場で、自分の大好きなテーマについて1〜2分間熱く語ってみましょう。",
        },
        {
          "id": "ST15",
          "title": "弱さの共有",
          "desc": "グループの前で「実はすごく緊張していたんだ」と素直に打ち明け、それを笑顔に変えてみましょう。",
        },
        {
          "id": "ST16",
          "title": "境界線の設定",
          "desc": "行きたくない誘いを受けたとき、言い訳や長すぎる説明をせずに、きっぱりと丁寧に断ってみましょう。",
        },
        {
          "id": "ST17",
          "title": "アクティブな仲裁役",
          "desc": "意見が対立している二人の間に入り、お互いが納得できる妥協点を見つける手助けをしてみましょう。",
        },
        {
          "id": "ST18",
          "title": "公の場での称賛",
          "desc": "誰かの努力や素晴らしい実績を、グループの全員の前で大声で称えてみましょう。",
        },
        {
          "id": "ST19",
          "title": "ストレートな依頼",
          "desc": "自分が尊敬する人に対して、アドバイスをもらうための時間やお願いを率直に依頼してみましょう。",
        },
        {
          "id": "ST20",
          "title": "会話の軌道修正",
          "desc": "退屈になってしまった会話の流れを、スムーズに全員が楽しめる興味深いテーマへと切り替えてみましょう。",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "ささやかな贈り物",
          "desc": "誰かに小さなお菓子などを差し出し、「あなたに合いそうだと思って」と言って渡してみましょう。",
        },
        {
          "id": "B2",
          "title": "大胆な提案",
          "desc": "小さなグループの人たちに、お出かけの具体的なプランや行ってみたい場所を提案してみましょう。",
        },
        {
          "id": "B3",
          "title": "深い感謝",
          "desc": "大切な人に対して、その人が自分の人生にいてくれることをどれほど感謝しているか、明確に伝えてみましょう。",
        },
        {
          "id": "B4",
          "title": "社交の主催者",
          "desc": "数人のために小さな集まりや、コーヒーを飲むお茶会を自分で企画して主催してみましょう。",
        },
        {
          "id": "B5",
          "title": "ディープダイブ",
          "desc": "誰かと一対一で、15分以上にわたって深く本質的なテーマについての対話を交わしてみましょう。",
        },
        {
          "id": "B6",
          "title": "高い壁への挑戦",
          "desc": "自分が「少し苦手だな」「気後れしてしまうな」と感じる相手に、自分から話しかけてみましょう。",
        },
        {
          "id": "B7",
          "title": "乾杯の音頭",
          "desc": "グループの集まりで、誰かを称えるための短い乾杯のスピーチや、ポジティブな紹介を行ってみましょう。",
        },
        {
          "id": "B8",
          "title": "しっかりとしたNO",
          "desc": "過度な要求に対して、言い訳をせずに、毅然としつつも優しい態度でしっかりと「お断り」を伝えてみましょう。",
        },
        {
          "id": "B9",
          "title": "メンターへの打診",
          "desc":
              "自分が憧れている人に対して、「10分だけお話させていただけませんか？」とアドバイスやメンターシップを依頼してみましょう。",
        },
        {
          "id": "B10",
          "title": "感情のリード",
          "desc": "親しい友人との間で、お互いの本音の感情やメンタルヘルスに関する深い相談の会話を切り出してみましょう。",
        },
        {
          "id": "B11",
          "title": "紛争の解決者",
          "desc": "身近な二人の間に起きた小さな衝突を、冷静な対話を促すことで解決へと導く手助けをしてみましょう。",
        },
        {
          "id": "B12",
          "title": "大胆なリスペクト",
          "desc": "完全に見知らぬ他人に対して、その人の素晴らしいと感じる部分を、ストレートに心から伝えてみましょう。",
        },
        {
          "id": "B13",
          "title": "プロへのアプローチ",
          "desc": "自分の専門分野や目指す業界のプロフェッショナルに自己紹介をし、キャリアのアドバイスを求めてみましょう。",
        },
        {
          "id": "B14",
          "title": "勇気ある真実",
          "desc": "関係性をより良くするために、伝えるのが少し気まずい、しかし相手のためになる真実を誠実に伝えてみましょう。",
        },
        {
          "id": "B15",
          "title": "満開のホスピタリティ",
          "desc": "自分で小さなイベントを企画し、来てくれたすべてのゲストが居心地よく過ごせるように気配りをしてみましょう。",
        },
        {
          "id": "B16",
          "title": "リーダーシップ",
          "desc": "会議やイベントの中で、自ら進行役やスピーカーを引き受けて一部分をリードしてみましょう。",
        },
        {
          "id": "B17",
          "title": "自己開示による励まし",
          "desc": "誰かを勇気づけるために、自分が過去に乗り越えた苦難や失敗の体験談をありのままに語ってみましょう。",
        },
        {
          "id": "B18",
          "title": "勇気ある謝罪",
          "desc": "ずっと過去のことだとしても、自分が昔してしまった過ちについて、自ら連絡を取って誠実に謝罪を伝えてみましょう。",
        },
        {
          "id": "B19",
          "title": "導き手",
          "desc": "自分よりも経験の浅い人に対して、自分の持っているスキルやノウハウを惜しみなく教えるサポートを申し出てみましょう。",
        },
        {
          "id": "B20",
          "title": "ソーシャル・アーキテクト",
          "desc": "友人グループのために、新しい定期的な集まりや、これから続く新しい「お決まりのイベント」の伝統を作ってみましょう。",
        },
      ],
    },
    'ko': {
      "Seedling": [
        {"id": "S1", "title": "첫 걸음", "desc": "오늘 한 사람과 눈을 맞추고 미소를 지어보세요."},
        {
          "id": "S2",
          "title": "소박한 인사",
          "desc": "이웃에게 '좋은 아침입니다' 또는 '안녕하세요'라고 말해보세요.",
        },
        {
          "id": "S3",
          "title": "감사의 표현",
          "desc": "가게 점원에게 명확하게 '감사합니다'라고 말해보세요.",
        },
        {
          "id": "S4",
          "title": "긍정적 관찰",
          "desc": "낯선 사람의 긍정적인 점을 발견하고 미소를 지어보세요.",
        },
        {
          "id": "S5",
          "title": "조용한 손인사",
          "desc": "멀리서 알아보는 사람에게 가볍게 손을 흔들어보세요.",
        },
        {
          "id": "S6",
          "title": "문 잡아주기",
          "desc": "내 뒤에 오는 사람을 위해 문을 열고 잠시 잡아주세요.",
        },
        {
          "id": "S7",
          "title": "가벼운 목례",
          "desc": "지나쳐 가는 동료에게 다정하게 고개를 끄덕여 인사해보세요.",
        },
        {
          "id": "S8",
          "title": "거울 앞 연습",
          "desc": "거울 앞에서 1분 동안 당신의 '자신감 넘치는 미소'를 연습해보세요.",
        },
        {
          "id": "S9",
          "title": "짧은 시선",
          "desc": "누군가를 2초 동안 바라본 뒤, 미소를 짓고 고개를 돌려보세요.",
        },
        {
          "id": "S10",
          "title": "조용한 칭찬",
          "desc": "누군가의 소셜 미디어 게시물에 따뜻한 댓글을 남겨보세요.",
        },
        {
          "id": "S11",
          "title": "공간 공유하기",
          "desc": "공공장소에서 누군가의 옆자리에 앉아 바로 시선을 피하지 않고 머물러보세요.",
        },
        {
          "id": "S12",
          "title": "짧은 양해",
          "desc": "복도에서 사람을 지나칠 때 정중하게 '실례합니다'라고 말해보세요.",
        },
        {
          "id": "S13",
          "title": "따뜻한 인사",
          "desc": "택배 기사님이나 배달원에게 '안녕하세요'라고 인사해보세요.",
        },
        {
          "id": "S14",
          "title": "작은 손짓",
          "desc": "아이 또는 (주인의 허락을 맡고) 반려동물에게 손을 흔들어보세요.",
        },
        {
          "id": "S15",
          "title": "부드러운 미소",
          "desc": "오늘 세 명의 서로 다른 사람들에게 미소를 지어보세요.",
        },
        {
          "id": "S16",
          "title": "아이컨택 챌린지",
          "desc": "계산원이 먼저 시선을 돌릴 때까지 눈을 맞추어보세요.",
        },
        {
          "id": "S17",
          "title": "차분한 호흡",
          "desc": "오늘 사교적인 공간에 들어가기 전에 크게 3번 심호흡을 해보세요.",
        },
        {
          "id": "S18",
          "title": "스마트폰 내려놓기",
          "desc": "사람이 많은 곳에서 스마트폰을 보지 않고 5분 동안 서 있어보세요.",
        },
        {
          "id": "S19",
          "title": "자연스러운 눈인사",
          "desc": "눈이 마주친 낯선 사람에게 가볍게 고개를 끄덕여보세요.",
        },
        {
          "id": "S20",
          "title": "상냥한 마무리",
          "desc": "가게를 나설 때 점원에게 '좋은 하루 보내세요'라고 말해보세요.",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "칭찬 건네기",
          "desc": "동료나 클래스메이트에게 진심 어린 칭찬을 해보세요.",
        },
        {"id": "SP2", "title": "질문하기", "desc": "낯선 사람에게 현재 시간이나 길을 물어보세요."},
        {
          "id": "SP3",
          "title": "스몰 토크",
          "desc": "누군가에게 '오늘 하루 어떻게 보내고 계세요?'라고 묻고 답변을 경청해보세요.",
        },
        {
          "id": "SP4",
          "title": "도움 요청",
          "desc": "매장 직원에게 특정 물건을 찾는 데 도움을 요청해보세요.",
        },
        {
          "id": "SP5",
          "title": "주문할 때 인사",
          "desc": "음료나 음식을 주문하며 직원에게 오늘 하루 어떠신지 가볍게 물어보세요.",
        },
        {
          "id": "SP6",
          "title": "먼저 다가가기",
          "desc": "주변에 있는 새로운 사람에게 자신을 소개해 보세요.",
        },
        {
          "id": "SP7",
          "title": "날씨 이야기",
          "desc": "줄을 서서 기다리는 동안 옆 사람에게 날씨 이야기를 건네보세요.",
        },
        {
          "id": "SP8",
          "title": "가벼운 관심",
          "desc": "직장 동료에게 '주말에 뭐 하셨어요?'라고 물어보세요.",
        },
        {
          "id": "SP9",
          "title": "도움의 손길",
          "desc": "어려움을 겪고 있는 듯한 사람을 보면 '좀 도와드릴까요?'라고 물어보세요.",
        },
        {
          "id": "SP10",
          "title": "의견 묻기",
          "desc": "친구에게 작은 물건을 보여주며 '이거 어때 보여?'라고 의견을 물어보세요.",
        },
        {
          "id": "SP11",
          "title": "사실 확인",
          "desc": "낯선 사람에게 무언가를 확인해 보세요 (예: '이 줄이 맞나요?').",
        },
        {
          "id": "SP12",
          "title": "공간에 대한 소감",
          "desc": "주변 환경에 대해 가벼운 한마디를 건네보세요 (예: '여기 사람 정말 많네요').",
        },
        {
          "id": "SP13",
          "title": "작은 부탁",
          "desc": "테이블에서 누군가에게 물건(예: 휴지)을 좀 건네달라고 부탁해보세요.",
        },
        {
          "id": "SP14",
          "title": "따뜻한 피드백",
          "desc": "식당을 나서기 전 직원에게 음식이 정말 맛있었다고 말해보세요.",
        },
        {
          "id": "SP15",
          "title": "안부 묻기",
          "desc": "한 달 동안 연락하지 않았던 사람에게 '잘 지내?'라고 안부 문자를 보내보세요.",
        },
        {
          "id": "SP16",
          "title": "열린 질문",
          "desc": "누군가에게 '이 도시에서 가장 좋아하는 장소가 어디예요?'라고 물어보세요.",
        },
        {
          "id": "SP17",
          "title": "가장 작은 용기",
          "desc": "낯선 사람에게 가장 가까운 화장실이 어디에 있는지 물어보세요.",
        },
        {
          "id": "SP18",
          "title": "소지품 칭찬",
          "desc": "누군가에게 신발/가방/액세서리가 멋지다고 말해보세요.",
        },
        {
          "id": "SP19",
          "title": "경청하기",
          "desc": "대답을 하기 전에 상대방이 말을 완전히 끝마칠 때까지 기다려보세요.",
        },
        {
          "id": "SP20",
          "title": "다정한 작별",
          "desc": "방금 짧은 대화를 나눈 사람에게 손を 흔들며 '다음에 봐요'라고 인사해보세요.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "의견 구하기",
          "desc": "누군가에게 책, 영화, 혹은 노래에 대한 의견을 물어보세요.",
        },
        {
          "id": "L2",
          "title": "꼬리 질문",
          "desc": "상대방이 자신에 대한 이야기를 한 후, 관련된 질문을 하나 더 이어가 보세요.",
        },
        {
          "id": "L3",
          "title": "추천 요청",
          "desc": "낯선 사람에게 이 근처에 맛있는 식당이 있는지 추천을 부탁해보세요.",
        },
        {
          "id": "L4",
          "title": "공통점 찾기",
          "desc": "누군가와 공통의 관심사를 찾고 그것에 대해 2분 동안 이야기해보세요.",
        },
        {
          "id": "L5",
          "title": "도움 제안",
          "desc": "누군가에게 가방을 들어주는 등의 작은 도움을 제안해보세요.",
        },
        {
          "id": "L6",
          "title": "상황 공유하기",
          "desc": "주변에서 일어나고 있는 일을 바탕으로 자연스럽게 대화를 시작해보세요.",
        },
        {
          "id": "L7",
          "title": "배경 질문",
          "desc": "누군가에게 '어떻게 이 일을 시작하게 되셨어요?'라고 물어보세요.",
        },
        {
          "id": "L8",
          "title": "경청과 요약",
          "desc": "상대방의 말을 중간에 끊지 않고 3분 동안 들은 후, 그 내용을 요약해서 말해보세요.",
        },
        {
          "id": "L9",
          "title": "웃음 공유",
          "desc": "소규모 그룹에게 짧고 재미있는 이야기나 농담을 건네보세요.",
        },
        {
          "id": "L10",
          "title": "호기심",
          "desc": "누군가에게 고향이 어디인지, 그리고 그곳의 어떤 점을 좋아하는지 물어보세요.",
        },
        {
          "id": "L11",
          "title": "진심 어린 관심",
          "desc": "직장 동료에게 회사 밖에서의 취미생활에 대해 물어보세요.",
        },
        {
          "id": "L12",
          "title": "다정한 팁",
          "desc": "당신이 잘하는 분야에 대해 누군가에게 도움이 되는 유용한 팁을 알려주세요.",
        },
        {
          "id": "L13",
          "title": "그룹 내 동의",
          "desc": "소규모 그룹 토론에서 누군가의 의견에 고개를 끄덕이며 동의를 표현해보세요.",
        },
        {
          "id": "L14",
          "title": "가벼운 초대",
          "desc": "누군가에게 '괜찮으시면 저희랑 같이 점심 드실래요?'라고 제안해보세요.",
        },
        {
          "id": "L15",
          "title": "진솔한 마음",
          "desc": "누군가에게 '전에 ~해주셨을 때 정말 감사했어요'라고 말하고 그 이유를 설명해보세요.",
        },
        {
          "id": "L16",
          "title": "질문 던지기",
          "desc": "누군가에게 '항상 궁금했는데, ~는 실제로 어떻게 작동하는 건가요?'라고 물어보세요.",
        },
        {
          "id": "L17",
          "title": "대화 유도",
          "desc": "그룹 안에서 2~3명이 함께 답변할 수 있을 만한 질문을 던져보세요.",
        },
        {
          "id": "L18",
          "title": "내면 칭찬",
          "desc": "상대방의 성격적 장점을 칭찬해보세요 (예: '얘기를 정말 잘 들어주시는 것 같아요').",
        },
        {
          "id": "L19",
          "title": "경험 공유",
          "desc": "대화 중에 '저도 그런 상황을 겪어본 적이 있어요'라고 공감을 표현해보세요.",
        },
        {
          "id": "L20",
          "title": "의도된 침묵",
          "desc": "대화 중에 찾아오는 어색한 침묵을 억지로 깨려 하지 않고 편안하게 받아들여 보세요.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "용기 있는 시작",
          "desc": "아직 잘 알지 못하는 사람에게 먼저 대화를 시도해보세요.",
        },
        {
          "id": "ST2",
          "title": "나의 이야기",
          "desc": "여러 사람이 모인 자리에서 자신의 짧은 이야기나 의견을 공유해보세요.",
        },
        {
          "id": "ST3",
          "title": "토론하기",
          "desc": "타인의 의견에 정중하게 이견을 제시하고 그 이유를 설명해보세요.",
        },
        {
          "id": "ST4",
          "title": "대화 합류",
          "desc": "진행 중인 그룹 대화에 자연스럽게 합류하여 생각을 담은 한 문장을 더해보세요.",
        },
        {"id": "ST5", "title": "화제 전환", "desc": "사교 모임에서 새로운 대화 주제를 먼저 꺼내보세요."},
        {
          "id": "ST6",
          "title": "공개적인 질문",
          "desc": "전체 회의나 수업 시간 중에 손을 들고 질문을 해보세요.",
        },
        {
          "id": "ST7",
          "title": "대담한 요청",
          "desc": "카페나 공원에서 낯선 사람에게 '여기 옆에 앉아도 될까요?'라고 물어보세요.",
        },
        {
          "id": "ST8",
          "title": "연결 다리",
          "desc": "서로 모르는 두 사람을 소개해 주고 그들의 공통점을 찾아 대화를 주선해보세요.",
        },
        {
          "id": "ST9",
          "title": "당당한 요구",
          "desc": "당신을 방해하거나 불편하게 하는 행동을 하는 사람에게 정중하게 멈춰달라고 요청해보세요.",
        },
        {
          "id": "ST10",
          "title": "스토리텔러",
          "desc": "3명 이상의 사람들 앞에서 자신이 주도하여 하나의 이야기를 흥미롭게 끝까지 들려줘 보세요.",
        },
        {
          "id": "ST11",
          "title": "부드러운 이의제기",
          "desc": "그룹 내의 일반적인 통념에 대해 정중하고 존중하는 태도로 의문을 제기해보세요.",
        },
        {
          "id": "ST12",
          "title": "먼저 하는 인사",
          "desc": "방에 들어설 때 그곳에 있는 모든 사람에게 가장 먼저 '안녕하세요!'라고 활기차게 인사해보세요.",
        },
        {
          "id": "ST13",
          "title": "공감의 귀",
          "desc": "누군가가 털어놓는 감정이나 불만을 끝까지 듣고 지지하는 답변을 건네보세요.",
        },
        {
          "id": "ST14",
          "title": "스피치 챌린지",
          "desc": "사교 모임에서 자신이 아주 좋아하는 주제에 대해 1~2분 동안 이야기해보세요.",
        },
        {
          "id": "ST15",
          "title": "솔직한 고백",
          "desc": "사람들 앞에서 '사실 제가 조금 긴장했었어요'라고 솔직히 고백하고 함께 웃어넘겨 보세요.",
        },
        {
          "id": "ST16",
          "title": "거절하기",
          "desc": "가고 싶지 않은 제안이나 초대를 받았을 때, 구태여 길게 변명하지 않고 정중하게 거절해보세요.",
        },
        {
          "id": "ST17",
          "title": "갈등 중재",
          "desc": "의견 대립이 있는 두 사람 사이에서 침착한 대화를 이끌어 절충안을 찾도록 도와보세요.",
        },
        {
          "id": "ST18",
          "title": "공개 칭찬",
          "desc": "여러 사람들 앞에서 누군가의 노력이나 성과를 소리 높여 칭찬해보세요.",
        },
        {
          "id": "ST19",
          "title": "직접적인 요청",
          "desc": "당신이 존경하는 사람에게 필요한 조언이나 제안을 직접적으로 구해보세요.",
        },
        {
          "id": "ST20",
          "title": "대화 이끌기",
          "desc": "지루해진 대화의 흐름을 자연스럽고 흥미진진한 다른 주제로 전환해 보세요.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "작은 선물",
          "desc": "누군가에게 작은 간식거리를 건네며 '너 생각나서 가져왔어'라고 말해보세요.",
        },
        {
          "id": "B2",
          "title": "아이디어 리드",
          "desc": "소규모 모임의 사람들에게 구체적인 약속 계획이나 방문하고 싶은 장소를 제안해보세요.",
        },
        {
          "id": "B3",
          "title": "깊은 감사",
          "desc": "가까운 사람에게 그 사람이 내 인생에 존재해 주어서 얼마나 감사한지 명확하게 표현해보세요.",
        },
        {
          "id": "B4",
          "title": "모임 주최",
          "desc": "몇 명의 사람들을 위해 작은 소모임이나 커피 약속을 직접 계획하고 주최해보세요.",
        },
        {
          "id": "B5",
          "title": "심층 대화",
          "desc": "누군가와 단둘이 15분이 넘는 시간 동안 깊고 진지한 인생의 주제로 이야기를 나누어보세요.",
        },
        {
          "id": "B6",
          "title": "두려움 마주하기",
          "desc": "스스로 조금 어렵게 느끼거나 긴장하게 만드는 상대에게 먼저 다가가 대화를 걸어보세요.",
        },
        {
          "id": "B7",
          "title": "축사 건네기",
          "desc": "모임 자리에서 누군가를 축하하거나 긍정적으로 소개해 주는 짧은 스피치를 해보세요.",
        },
        {
          "id": "B8",
          "title": "확실한 거절",
          "desc": "무리한 요구에 대해 장황하게 설명하지 않고 단호하면서도 다정한 태도로 거절의 선을 그어보세요.",
        },
        {
          "id": "B9",
          "title": "멘토링 제안",
          "desc": "존경하는 분에게 '10분만 대화를 나누거나 조언을 얻을 수 있을까요?'라고 멘토링을 요청해보세요.",
        },
        {
          "id": "B10",
          "title": "감정 교류",
          "desc": "친한 친구와 속마음의 감정이나 마음 건강에 관한 깊은 고민 상담을 먼저 시작해보세요.",
        },
        {
          "id": "B11",
          "title": "소통의 중재자",
          "desc": "가까운 사람들 간의 작은 오해나 갈등을 차분한 대화의 자리를 마련해 해결하도록 도와주세요.",
        },
        {
          "id": "B12",
          "title": "과감한 리سبكت",
          "desc": "완전한 타인에게 그 사람에 대해 진심으로 감탄하거나 존경스러운 부분을 스트레이트로 전달해보세요.",
        },
        {
          "id": "B13",
          "title": "네트워킹",
          "desc": "자신의 전문 분야나 목표로 하는 직종의 전문가에게 자신을 소개하고 조언을 구해보세요.",
        },
        {
          "id": "B14",
          "title": "용기 있는 진실",
          "desc": "관계를 더 건강하게 발전시키기 위해, 꺼내기 어렵지만 도움이 되는 진실을 성실하게 말해보세요.",
        },
        {
          "id": "B15",
          "title": "완벽한 환대",
          "desc": "작은 소셜 이벤트를 열고 찾아와 준 모든 게스트가 환영받는다고 느끼도록 배려해보세요.",
        },
        {
          "id": "B16",
          "title": "리더십 발휘",
          "desc": "회의나 행사 진행 중에 스스로 발표자나 리더 역할을 맡아 일부분을 이끌어보세요.",
        },
        {
          "id": "B17",
          "title": "약점 공유하기",
          "desc": "타인에게 위로와 용기를 주기 위해 자신이 극복했던 상처나 극복의 경험담을 숨김없이 들려줘 보세요.",
        },
        {
          "id": "B18",
          "title": "진심 어린 사과",
          "desc": "비록 아주 오래전의 일일지라도 내가 과거에 했던 실수에 대해 스스로 먼저 연락하여 진심으로 사과해보세요.",
        },
        {
          "id": "B19",
          "title": "멘토 되기",
          "desc": "나보다 경험이 적은 사람에게 내 스킬이나 노하우를 아낌없이 나누어 주는 멘토 역할을 자처해보세요.",
        },
        {
          "id": "B20",
          "title": "소셜 아키텍트",
          "desc": "친구 모임을 위해 매번 반복될 수 있는 새로운 약속 전통이나 고정적인 만남 행사를 만들어보세요.",
        },
      ],
    },
    'zh': {
      "Seedling": [
        {"id": "S1", "title": "第一步", "desc": "今天，与一个人进行眼神接触并报以微笑。"},
        {"id": "S2", "title": "简单的问候", "desc": "向邻居说一声“早上好”或“你好”。"},
        {"id": "S3", "title": "道谢", "desc": "清晰地对店员说一声“谢谢”。"},
        {"id": "S4", "title": "观察", "desc": "注意到陌生人身上的某个闪光点，并报以微笑。"},
        {"id": "S5", "title": "静静地挥手", "desc": "远远地向你认出的一位熟人挥手致意。"},
        {"id": "S6", "title": "帮人留门", "desc": "为身后的路人把住门，等他们走过去。"},
        {"id": "S7", "title": "点头示意", "desc": "与同事擦肩而过时，友好地向对方点头致意。"},
        {"id": "S8", "title": "镜子练习", "desc": "对着镜子练习你“自信的微笑”，持续1分钟。"},
        {"id": "S9", "title": "短暂一瞥", "desc": "注视某人2秒钟，然后微笑并移开视线。"},
        {"id": "S10", "title": "默默称赞", "desc": "在别人的社交媒体动态下写一条善意的评论。"},
        {"id": "S11", "title": "共享空间", "desc": "在公共区域坐在某人身边，且不要立刻转过头去。"},
        {"id": "S12", "title": "简单的礼貌", "desc": "在走廊通过某人身边时，礼貌地说一声“借过”或“抱歉”。"},
        {"id": "S13", "title": "温暖的问候", "desc": "向快递员或外卖小哥说一声“你好”或“辛苦了”。"},
        {"id": "S14", "title": "小小的挥手", "desc": "向一个孩子或宠物挥挥手（需征得主人同意）。"},
        {"id": "S15", "title": "温柔的微笑", "desc": "今天向三位不同的路人展现温柔的微笑。"},
        {"id": "S16", "title": "眼神接触挑战", "desc": "与收银员保持眼神接触，直到对方先移开视线。"},
        {"id": "S17", "title": "调整呼吸", "desc": "今天在进入任何社交场所之前，进行3次深呼吸。"},
        {"id": "S18", "title": "活在当下", "desc": "在人群密集的地方站立5分钟，期间不看一眼手机。"},
        {"id": "S19", "title": "自然地点头", "desc": "向与你有眼神接触的陌生人自然地点头示意。"},
        {"id": "S20", "title": "轻声祝愿", "desc": "离开商店时，对店员说一声“祝你今天过得愉快”。"},
      ],
      "Sprout": [
        {"id": "SP1", "title": "称赞他人", "desc": "真诚地赞美一位同事或同学。"},
        {"id": "SP2", "title": "询问", "desc": "向陌生人询问时间或问路。"},
        {"id": "SP3", "title": "闲聊", "desc": "问某人“你今天过得怎么样？”并认真倾听他们的回答。"},
        {"id": "SP4", "title": "寻求协助", "desc": "请店员帮忙寻找某件特定的商品。"},
        {"id": "SP5", "title": "点单互动", "desc": "在点饮料或食物时，顺便问候一下店员过得怎么样。"},
        {"id": "SP6", "title": "结识新朋友", "desc": "向你生活圈子里的某个新面孔主动做自我介绍。"},
        {"id": "SP7", "title": "聊聊天气", "desc": "在排队等待时，和旁边的人随口聊两句天气。"},
        {"id": "SP8", "title": "简单关心", "desc": "问同事一句“你周末过得怎么样？”"},
        {
          "id": "SP9",
          "title": "主动提供帮助",
          "desc": "如果看到有人似乎遇到了麻烦，主动上前询问“需要帮忙吗？”。",
        },
        {"id": "SP10", "title": "征求意见", "desc": "拿一个随身的小物件问朋友：“你觉得这个怎么样？”。"},
        {"id": "SP11", "title": "确认细节", "desc": "向陌生人确认一件事情（例如：“请问是这里在排队吗？”）。"},
        {
          "id": "SP12",
          "title": "环境评价",
          "desc": "对周围的环境随口发表一句评价（例如：“今天这儿真是人山人海”）。",
        },
        {"id": "SP13", "title": "举手之劳", "desc": "在餐桌上请别人帮你递一下东西（比如一张纸巾）。"},
        {"id": "SP14", "title": "热情的反馈", "desc": "离开餐厅前，真诚地赞美服务员“今天的菜品棒极了”。"},
        {
          "id": "SP15",
          "title": "偶尔的联络",
          "desc": "给一个已经断联近一个月的朋友发一条“最近怎么样？”的问候短信。",
        },
        {"id": "SP16", "title": "开放式提问", "desc": "问某人：“这座城市里你最推荐去哪里玩？”。"},
        {"id": "SP17", "title": "微小的冒险", "desc": "向陌生人打听离这儿最近的洗手间在哪里。"},
        {"id": "SP18", "title": "赞美配饰", "desc": "赞美别人的鞋子/包包/配饰很好看。"},
        {"id": "SP19", "title": "礼貌的倾听", "desc": "在回应别人之前，耐心等待对方把话完全说完。"},
        {"id": "SP20", "title": "友好地道别", "desc": "向刚刚进行过短暂交谈的人挥手并微笑着说“再见”。"},
      ],
      "Leaf": [
        {"id": "L1", "title": "探询观点", "desc": "询问某人对某本书、某部电影或某首歌曲的看法。"},
        {
          "id": "L2",
          "title": "拓展提问",
          "desc": "在对方分享了关于他们自己的事情后，顺着话题追问一个相关的问题。",
        },
        {"id": "L3", "title": "寻求推荐", "desc": "向陌生人打听这附近有没有什么值得一试的美食餐厅。"},
        {"id": "L4", "title": "寻找共鸣", "desc": "找出你与某人的共同兴趣，并围绕这个话题畅聊2分钟。"},
        {"id": "L5", "title": "搭一把手", "desc": "主动提出帮别人做一件小事（比如帮人提提包）。"},
        {"id": "L6", "title": "就地取材", "desc": "根据你们身边正在发生的事情，自然地开启一段对话。"},
        {"id": "L7", "title": "深度提问", "desc": "问某人：“您当初是怎么进入这行工作的呢？”。"},
        {
          "id": "L8",
          "title": "积极倾听",
          "desc": "专心倾听对方讲述3分钟而不打断，随后简单复述或总结一下他们的话。",
        },
        {"id": "L9", "title": "分享快乐", "desc": "向一小群人讲一个幽默的短篇故事或开个小玩笑。"},
        {"id": "L10", "title": "保持好奇", "desc": "询问某人来自哪里，以及他们最喜欢家乡的什么地方。"},
        {"id": "L11", "title": "由衷的兴趣", "desc": "向同事了解一下他们工作之外的兴趣爱好。"},
        {"id": "L12", "title": "倾囊相助", "desc": "就你非常擅长的某项技能，给别人分享一个实用的小技巧。"},
        {"id": "L13", "title": "群体附和", "desc": "在小组讨论中，对某人发表的观点明确表示赞同或点头。"},
        {"id": "L14", "title": "随口相邀", "desc": "对某人说：“要不要和我们一起去吃午饭？”。"},
        {
          "id": "L15",
          "title": "真诚反馈",
          "desc": "真诚地对某人说：“你之前做X这件事时我真的很感激”，并解释理由。",
        },
        {"id": "L16", "title": "探索未知", "desc": "向人请教：“我一直很好奇，X到底是怎么运作的呀？”。"},
        {"id": "L17", "title": "抛砖引玉", "desc": "在团队里抛出一个需要两三个人共同探讨或补充回答的问题。"},
        {
          "id": "L18",
          "title": "夸赞性格",
          "desc": "赞美某人的性格闪光点（例如：“你真的是一个非常棒的倾听者”）。",
        },
        {"id": "L19", "title": "共鸣分享", "desc": "在谈话中说一句：“我也曾感同身受过那样的处境”。"},
        {"id": "L20", "title": "接纳留白", "desc": "允许对话中出现片刻的安静，而不用刻意地寻找话茬去填补它。"},
      ],
      "Stem": [
        {"id": "ST1", "title": "勇敢试水", "desc": "主动与一个你不太熟悉的人开始交谈。"},
        {"id": "ST2", "title": "坦诚分享", "desc": "在集体环境里公开分享一段个人的小故事或表达观点。"},
        {"id": "ST3", "title": "良性交锋", "desc": "礼貌地对某人的观点表达不同意见，并清晰解释原因。"},
        {"id": "ST4", "title": "融入圈子", "desc": "自然地加入一段正在进行的多人对话，并给出一个有见地的观点。"},
        {"id": "ST5", "title": "开辟话题", "desc": "在一个社交圈子里主动发起一个全新的聊天主题。"},
        {"id": "ST6", "title": "公开发言", "desc": "在公开会议或者课堂环境中，举手提出一个问题。"},
        {"id": "ST7", "title": "大胆询问", "desc": "在咖啡厅或公园里问陌生人：“请问我可以坐在这旁边吗？”。"},
        {"id": "ST8", "title": "社交桥梁", "desc": "把两个互不相识的朋友介绍给彼此，并帮他们找到一个共同话题。"},
        {
          "id": "ST9",
          "title": "合理维权",
          "desc": "当某人的行为打扰到你时，礼貌地请对方挪一下位置或停止该行为。",
        },
        {"id": "ST10", "title": "故事主角", "desc": "在三个及以上的人面前，由你主导并完整叙述一段故事。"},
        {
          "id": "ST11",
          "title": "友好博弈",
          "desc": "用一种非常友好且充满敬意的方式，在群体中对某个大众观点提出质疑。",
        },
        {"id": "ST12", "title": "主动破冰", "desc": "进入房间时，成为全场第一个主动向所有人打招呼问好的人。"},
        {
          "id": "ST13",
          "title": "共情接纳",
          "desc": "耐心地倾听某人倾诉心中的委屈或负面情绪，并给出支持性的回应。",
        },
        {
          "id": "ST14",
          "title": "即兴演说",
          "desc": "在社交聚会上，围绕一个你非常热爱的话题即兴畅聊1-2分钟。",
        },
        {
          "id": "ST15",
          "title": "展露脆弱",
          "desc": "在一群人面前坦白承认自己之前其实对某事很紧张，然后一笑了之。",
        },
        {"id": "ST16", "title": "建立边界", "desc": "礼貌地拒绝一个你不想参加的邀请，且无需找借口过度解释。"},
        {"id": "ST17", "title": "积极调解", "desc": "在两个出现分歧的人之间居中调解，帮助他们找到折中办法。"},
        {"id": "ST18", "title": "当众表扬", "desc": "在一群人面前公开赞扬某人的努力或取得的成就。"},
        {"id": "ST19", "title": "直接触达", "desc": "直接向你仰慕的某个人请求一项帮助或寻求实质性的建议。"},
        {
          "id": "ST20",
          "title": "掌控风向",
          "desc": "当聊天话题变得有些沉闷时，丝滑地将其引导至另一个有趣的领域。",
        },
      ],
      "Bloom": [
        {"id": "B1", "title": "投桃报李", "desc": "递给别人一份小甜点或小礼物，并说：“我觉得你会喜欢这个”。"},
        {"id": "B2", "title": "带头做决定", "desc": "向一小群人主动提议一项具体的聚会活动计划或出游地点。"},
        {
          "id": "B3",
          "title": "致谢生命",
          "desc": "真诚且具体地告诉生命中重要的人，为什么你如此庆幸生活中能有他们的陪伴。",
        },
        {"id": "B4", "title": "社交组织者", "desc": "亲自牵头为几位朋友策划并组织一场小型聚会或下午茶约会。"},
        {
          "id": "B5",
          "title": "深度对话",
          "desc": "与某人单独进行一场超过15分钟的、触及灵魂与人生意义的深度长谈。",
        },
        {
          "id": "B6",
          "title": "打破壁垒",
          "desc": "主动找一个让你觉得有些威严、难以接近或者让你感到露怯的人聊天。",
        },
        {"id": "B7", "title": "当众致辞", "desc": "在集体聚会上，自发站出来说几句赞美某人或者祝福大家的祝酒词。"},
        {
          "id": "B8",
          "title": "坚定的“不”",
          "desc": "面对不合理的请求，态度温和但语气坚定地予以拒绝，且不找借口粉饰。",
        },
        {
          "id": "B9",
          "title": "引路人邀请",
          "desc": "向你仰慕的业内前辈发出诚挚邀请，请求占用对方10分钟时间取取经或建立指导关系。",
        },
        {
          "id": "B10",
          "title": "情感宣泄",
          "desc": "主动在好朋友面前坦露真心，开启关于彼此情感状态或心理健康的深度沟通。",
        },
        {
          "id": "B11",
          "title": "矛盾化解者",
          "desc": "引导陷入冷战或小冲突的两个人坐下来，通过理性的沟通冰释前嫌。",
        },
        {
          "id": "B12",
          "title": "大声说出赞美",
          "desc": "迎面走向一个完全陌生的路人，大方真诚地夸赞对方身上让你由衷佩服的一点。",
        },
        {
          "id": "B13",
          "title": "拓展职业圈",
          "desc": "主动向你所在行业的精英人士做自我介绍，并大方地向对方请教职业建议。",
        },
        {
          "id": "B14",
          "title": "良药苦口",
          "desc": "为了让彼此的关系长久健康，鼓起勇气跟对方说出一句虽然刺耳但对大局有利的真心话。",
        },
        {
          "id": "B15",
          "title": "完美东道主",
          "desc": "自己张罗一场小型的社交活动，并尽全力确保现场到访的每一位客人都感到受重视。",
        },
        {
          "id": "B16",
          "title": "领头羊",
          "desc": "在大型会议或团建活动中，自荐担任主持人或负责牵头组织其中一个环节。",
        },
        {
          "id": "B17",
          "title": "伤疤的勋章",
          "desc": "向他人毫无保留地坦白自己曾经熬过来的低谷或失败经历，以此去激励和照亮别人。",
        },
        {
          "id": "B18",
          "title": "迟到的道歉",
          "desc": "主动联系某人，为自己很久以前做过的一件错事真诚地表达歉意，哪怕已经时过境迁。",
        },
        {
          "id": "B19",
          "title": "导师角色",
          "desc": "主动向比你经验浅的新人提供帮助，毫无保留地向他们传授你的某项专业核心技能。",
        },
        {
          "id": "B20",
          "title": "社交架构师",
          "desc": "为你的朋友们开创一项全新的固定小传统（例如每周固定聚会或周期性主题碰头会）。",
        },
      ],
    },
    'it': {
      "Seedling": [
        {
          "id": "S1",
          "title": "Il Primo Passo",
          "desc": "Cerca il contatto visivo e sorridi a una persona oggi.",
        },
        {
          "id": "S2",
          "title": "Un Semplice Saluto",
          "desc": "Dì 'Buongiorno' o 'Ciao' a un vicino di casa.",
        },
        {
          "id": "S3",
          "title": "Il Ringraziamento",
          "desc": "Dì 'Grazie' in modo chiaro a un negoziante.",
        },
        {
          "id": "S4",
          "title": "L'Osservazione",
          "desc": "Nota qualcosa di positivo in uno sconosciuto e sorridi.",
        },
        {
          "id": "S5",
          "title": "Il Ciao Silenzioso",
          "desc":
              "Fai un cenno con la mano a qualcuno che riconosci da lontano.",
        },
        {
          "id": "S6",
          "title": "Tieni la Porta",
          "desc": "Tieni la porta aperta per qualcuno dietro di te.",
        },
        {
          "id": "S7",
          "title": "Il Cenno",
          "desc":
              "Fai un cenno amichevole con la testa a un collega mentre gli passi vicino.",
        },
        {
          "id": "S8",
          "title": "Lo Specchio",
          "desc":
              "Esercitati con il tuo 'sorriso sicuro' allo specchio per 1 minuto.",
        },
        {
          "id": "S9",
          "title": "Lo Sguardo Breve",
          "desc":
              "Guarda qualcuno per 2 secondi, poi sorridi e distogli lo sguardo.",
        },
        {
          "id": "S10",
          "title": "La Lode Silenziosa",
          "desc":
              "Scrivi un bel commento sotto il post sui social di qualcuno.",
        },
        {
          "id": "S11",
          "title": "Spazio Condiviso",
          "desc":
              "Siediti accanto a qualcuno in un'area pubblica senza distogliere subito lo sguardo.",
        },
        {
          "id": "S12",
          "title": "Il Semplice Riconoscimento",
          "desc":
              "Dì 'Permesso' o 'Scusa' educatamente quando passi accanto a qualcuno in un corridoio.",
        },
        {
          "id": "S13",
          "title": "Il Saluto Caloroso",
          "desc": "Dì 'Ciao' a un corriere o a un fattorino.",
        },
        {
          "id": "S14",
          "title": "Il Piccolo Cenno",
          "desc":
              "Fai un cenno con la mano a un bambino o a un animale domestico (con il permesso del proprietario).",
        },
        {
          "id": "S15",
          "title": "Il Sorriso Gentile",
          "desc": "Sorridi a tre persone diverse oggi.",
        },
        {
          "id": "S16",
          "title": "Sfida del Contatto Visivo",
          "desc":
              "Mantieni il contatto visivo con un cassiere finché non distoglie lo sguardo per primo.",
        },
        {
          "id": "S17",
          "title": "Il Respiro Calmo",
          "desc":
              "Fai 3 respiri profondi prima di entrare in uno spazio sociale oggi.",
        },
        {
          "id": "S18",
          "title": "La Presenza",
          "desc":
              "Rimani in un'area affollata per 5 minutes senza guardare il telefono.",
        },
        {
          "id": "S19",
          "title": "Il Cenno Casuale",
          "desc":
              "Fai un cenno con la testa a uno sconosciuto che incrocia il tuo sguardo.",
        },
        {
          "id": "S20",
          "title": "La Voce Dolce",
          "desc": "Dì 'Buona giornata' a qualcuno mentre esci da un negozio.",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "Il Complimento",
          "desc":
              "Fai un complimento sincero a un collega o a un compagno di classe.",
        },
        {
          "id": "SP2",
          "title": "La Domanda",
          "desc": "Chiedi l'ora o indicazioni stradali a uno sconosciuto.",
        },
        {
          "id": "SP3",
          "title": "Small Talk",
          "desc":
              "Chiedi a qualcuno 'Come va la giornata?' e ascolta la risposta.",
        },
        {
          "id": "SP4",
          "title": "La Richiesta",
          "desc":
              "Chiedi aiuto a un dipendente di un negozio per trovare un articolo specifico.",
        },
        {
          "id": "SP5",
          "title": "L'Ordinazione",
          "desc":
              "Ordina da bere o da mangiare e chiedi al personale come sta.",
        },
        {
          "id": "SP6",
          "title": "Il Saluto iniziale",
          "desc": "Presentati a qualcuno di nuovo nella tua zona.",
        },
        {
          "id": "SP7",
          "title": "Parlare del Meteo",
          "desc": "Fai un commento sul meteo a qualcuno mentre sei in coda.",
        },
        {
          "id": "SP8",
          "title": "La Semplice Richiesta",
          "desc": "Chiedi a un collega 'Cosa hai fatto nel fine settimana?'",
        },
        {
          "id": "SP9",
          "title": "L'Offerta d'Aiuto",
          "desc":
              "Chiedi a qualcuno 'Ti serve una mano?' se sembra in difficoltà.",
        },
        {
          "id": "SP10",
          "title": "L'Opinione",
          "desc":
              "Chiedi a un amico 'Cosa ne pensi di questo?' riguardo a un piccolo oggetto.",
        },
        {
          "id": "SP11",
          "title": "La Conferma",
          "desc":
              "Conferma un dettaglio con uno sconosciuto (es. 'È questa la fila giusta?').",
        },
        {
          "id": "SP12",
          "title": "Lo Spazio Condiviso",
          "desc":
              "Fai un piccolo commento sull'ambiente circostante (es. 'C'è davvero molta gente').",
        },
        {
          "id": "SP13",
          "title": "Il Piccolo Favore",
          "desc":
              "Chiedi a qualcuno di passarti qualcosa (come un tovagliolo) a tavola.",
        },
        {
          "id": "SP14",
          "title": "Il Feedback Caloroso",
          "desc":
              "Dì a un cameriere che il cibo era ottimo prima di andare via.",
        },
        {
          "id": "SP15",
          "title": "Il Controllo Casuale",
          "desc":
              "Invia un messaggio con scritto 'Come stai?' a qualcuno con cui non parli da un mese.",
        },
        {
          "id": "SP16",
          "title": "La Domanda Aperta",
          "desc":
              "Chiedi a qualcuno 'Qual è il tuo posto preferito da visitare in questa città?'",
        },
        {
          "id": "SP17",
          "title": "Il Rischio Minimo",
          "desc":
              "Chiedi a uno sconosciuto se sa dove si trova il bagno più vicino.",
        },
        {
          "id": "SP18",
          "title": "Elogio di un Oggetto",
          "desc":
              "Dì a qualcuno che ti piacciono le sue scarpe/borsa/accessorio.",
        },
        {
          "id": "SP19",
          "title": "La Pausa Educata",
          "desc":
              "Aspetta che qualcuno abbia finito completamente di parlare prima di rispondergli.",
        },
        {
          "id": "SP20",
          "title": "Il Saluto Amichevole",
          "desc":
              "Fai un cenno con la mano e dì 'Ciao' a qualcuno con cui hai appena avuto una breve interazione.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "Cercatore di Opinioni",
          "desc":
              "Chiedi a qualcuno un parere su un libro, un film o una canzone.",
        },
        {
          "id": "L2",
          "title": "Il Dettaglio",
          "desc":
              "Fai una domanda di approfondimento dopo che qualcuno ti ha raccontato qualcosa di sé.",
        },
        {
          "id": "L3",
          "title": "La Raccomandazione",
          "desc":
              "Chiedi a uno sconosciuto un consiglio su un buon posto dove mangiare nelle vicinanze.",
        },
        {
          "id": "L4",
          "title": "La Connessione",
          "desc":
              "Trova un interesse comune con qualcuno e parlane per 2 minuti.",
        },
        {
          "id": "L5",
          "title": "La Mano Aiuto",
          "desc":
              "Offriti di aiutare qualcuno con un piccolo compito (come portare una borsa).",
        },
        {
          "id": "L6",
          "title": "L'Osservazione Sociale",
          "desc":
              "Inizia una conversazione basandoti su qualcosa che sta accadendo intorno a entrambi.",
        },
        {
          "id": "L7",
          "title": "La Domanda Aperta",
          "desc": "Chiedi a qualcuno 'Come hai iniziato a fare questo lavoro?'",
        },
        {
          "id": "L8",
          "title": "L'Ascoltatore Attivo",
          "desc":
              "Ascolta qualcuno per 3 minuti senza interromperlo, poi riassumi quello che ha detto.",
        },
        {
          "id": "L9",
          "title": "La Risata Condivisa",
          "desc":
              "Racconta una storia breve e divertente o una barzelletta a un piccolo gruppo.",
        },
        {
          "id": "L10",
          "title": "La Curiosità",
          "desc": "Chiedi a qualcuno di dov'è e cosa gli piace di quel posto.",
        },
        {
          "id": "L11",
          "title": "L'Interesse Sincero",
          "desc":
              "Chiedi a un collega quali sono i suoi hobby fuori dal lavoro.",
        },
        {
          "id": "L12",
          "title": "Il Consiglio Delicato",
          "desc":
              "Dai a qualcuno un consiglio utile su qualcosa in cui sei bravo.",
        },
        {
          "id": "L13",
          "title": "Il Cenno di Gruppo",
          "desc":
              "Esprimi accordo con il punto di vista di qualcuno in una discussione di gruppo.",
        },
        {
          "id": "L14",
          "title": "L'Invito Informale",
          "desc": "Chiedi a qualcuno 'Ti andrebbe di unirti a noi per pranzo?'",
        },
        {
          "id": "L15",
          "title": "La Riflessione Onesta",
          "desc":
              "Dì a qualcuno 'Ho davvero apprezzato quando hai fatto X' e spiega il perché.",
        },
        {
          "id": "L16",
          "title": "Il Divario di Curiosità",
          "desc":
              "Chiedi a qualcuno 'Mi sono sempre chiesto, come funziona effettivamente X?'",
        },
        {
          "id": "L17",
          "title": "La Guida del Piccolo Gruppo",
          "desc":
              "Fai una domanda che richieda la risposta di 2 o 3 persone in un gruppo.",
        },
        {
          "id": "L18",
          "title": "Il Complimento Autentico",
          "desc":
              "Fai un complimento a qualcuno su un tratto della sua personalità (es. 'Sei un ottimo ascoltatore').",
        },
        {
          "id": "L19",
          "title": "L'Esperienza Condivisa",
          "desc":
              "Dì 'Mi sono trovato anch'io in quella situazione' durante una conversazione.",
        },
        {
          "id": "L20",
          "title": "La Pausa Significativa",
          "desc":
              "Permetti che si crei un momento di silenzio in una conversazione senza correre a riempirlo.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "L'Inizio Coraggioso",
          "desc": "Avvia una conversazione con qualcuno che non conosci bene.",
        },
        {
          "id": "ST2",
          "title": "La Condivisione Onesta",
          "desc":
              "Condividi una piccola storia personale o un'opinione in un contesto di gruppo.",
        },
        {
          "id": "ST3",
          "title": "Il Dibattito",
          "desc":
              "Disapprova educatamente l'opinione di qualcuno e spiega il perché.",
        },
        {
          "id": "ST4",
          "title": "L'Ingresso nel Gruppo",
          "desc":
              "Unisciti a una conversazione di gruppo e contribuisci con una frase ponderata.",
        },
        {
          "id": "ST5",
          "title": "La Guida del Tema",
          "desc":
              "Introduci un nuovo argomento di conversazione in un gruppo sociale.",
        },
        {
          "id": "ST6",
          "title": "La Domanda Pubblica",
          "desc":
              "Fai una domanda in una riunione pubblica o in un contesto scolastico.",
        },
        {
          "id": "ST7",
          "title": "La Richiesta Audace",
          "desc":
              "Chiedi a uno sconosciuto se puoi sederti accanto a lui in un bar o in un parco.",
        },
        {
          "id": "ST8",
          "title": "Il Ponte di Conversazione",
          "desc":
              "Presenta due persone che non si conoscono e trova un punto in comune.",
        },
        {
          "id": "ST9",
          "title": "Il Bisogno Assertivo",
          "desc":
              "Chiedi educatamente a qualcuno di spostarsi o di smettere di fare qualcosa che ti dà fastidio.",
        },
        {
          "id": "ST10",
          "title": "Il Narratore",
          "desc":
              "Prendi l'iniziativa nel raccontare una storia a un gruppo di 3 o più persone.",
        },
        {
          "id": "ST11",
          "title": "La Sfida Aperta",
          "desc":
              "Metti in discussione un'opinione comune in un gruppo in modo amichevole e rispettoso.",
        },
        {
          "id": "ST12",
          "title": "L'Iniziativa Sociale",
          "desc":
              "Sii la prima persona a dire 'Ciao' a tutti quando entri in una stanza.",
        },
        {
          "id": "ST13",
          "title": "L'Ascolto Empatico",
          "desc":
              "Ascolta qualcuno che si sta sfogando e offri una risposta di supporto.",
        },
        {
          "id": "ST14",
          "title": "La Presentazione Pubblica",
          "desc":
              "Parla per 1-2 minuti di un argomento che ami in un incontro sociale.",
        },
        {
          "id": "ST15",
          "title": "La Condivisione Vulnerabile",
          "desc":
              "Ammetti davanti a un gruppo che eri nervoso per qualcosa, e fateci una risata sopra.",
        },
        {
          "id": "ST16",
          "title": "L'Impostazione dei Confini",
          "desc":
              "Rifiuta educatamente un invito a cui non vuoi partecipare senza dare troppe spiegazioni.",
        },
        {
          "id": "ST17",
          "title": "Il Mediatore Attivo",
          "desc":
              "Aiuta due persone a trovare una via di mezzo in un disaccordo.",
        },
        {
          "id": "ST18",
          "title": "Il Complimento Pubblico",
          "desc":
              "Loda pubblicamente lo sforzo o il risultato di qualcuno in un gruppo.",
        },
        {
          "id": "ST19",
          "title": "L'Approccio Diretto",
          "desc":
              "Chiedi direttamente a qualcuno un favore o un consiglio di cui hai bisogno.",
        },
        {
          "id": "ST20",
          "title": "Il Pivot della Conversazione",
          "desc":
              "Fai passare fluidamente una conversazione da un argomento noioso a uno interessante.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "Il Regalo",
          "desc":
              "Fai un piccolo pensiero a qualcuno e dì 'Ho pensato che ti sarebbe piaciuto'.",
        },
        {
          "id": "B2",
          "title": "La Guida Audace",
          "desc":
              "Proponi un programma o un posto da visitare a un piccolo gruppo di persone.",
        },
        {
          "id": "B3",
          "title": "L'Apprezzamento",
          "desc":
              "Dì a qualcuno nello specifico perché apprezzi averlo nella tua vita.",
        },
        {
          "id": "B4",
          "title": "L'Ospite Sociale",
          "desc":
              "Organizza un piccolo incontro o un appuntamento per un caffè per poche persone.",
        },
        {
          "id": "B5",
          "title": "L'Immersione Profonda",
          "desc":
              "Fai una conversazione profonda e significativa con qualcuno per più di 15 minuti.",
        },
        {
          "id": "B6",
          "title": "Il Picco di Fiducia",
          "desc":
              "Inizia una conversazione con qualcuno che trovi intimidatorio.",
        },
        {
          "id": "B7",
          "title": "Il Brindisi Pubblico",
          "desc":
              "Fai un breve brindisi positivo o un elogio a qualcuno in un gruppo.",
        },
        {
          "id": "B8",
          "title": "L'Impostore di Confini",
          "desc":
              "Dì 'No' a una richiesta in modo fermo ma gentile, senza dare troppe spiegazioni.",
        },
        {
          "id": "B9",
          "title": "La Richiesta Diretta",
          "desc":
              "Chiedi a qualcuno che ammiri una chiacchierata di 10 minuti o un consiglio.",
        },
        {
          "id": "B10",
          "title": "La Guida Emotiva",
          "desc":
              "Avvia una conversazione sui sentimenti o sulla salute mentale con un amico.",
        },
        {
          "id": "B11",
          "title": "Il Mediatore Sociale",
          "desc":
              "Aiuta due persone a risolvere un piccolo conflitto attraverso una conversazione calma.",
        },
        {
          "id": "B12",
          "title": "Il Complimento Audace",
          "desc":
              "Dì a un perfetto sconosciuto qualcosa che ammiri sinceramente di lui.",
        },
        {
          "id": "B13",
          "title": "La Mossa di Networking",
          "desc":
              "Presentati a un professionista del tuo settore e chiedigli un consiglio.",
        },
        {
          "id": "B14",
          "title": "La Verità Coraggiosa",
          "desc":
              "Dì a qualcuno una verità difficile ma utile per il vostro rapporto.",
        },
        {
          "id": "B15",
          "title": "La Piena Fioritura",
          "desc":
              "Organizza un piccolo evento sociale e assicurati che ogni ospite si senta il benvenuto.",
        },
        {
          "id": "B16",
          "title": "Il Oratore Pubblico",
          "desc":
              "Offriti volontario per parlare o guidare una piccola parte di una riunione o di un evento.",
        },
        {
          "id": "B17",
          "title": "La Guida Vulnerabile",
          "desc":
              "Condividi una difficoltà che hai superato per incoraggiare qualcun altro.",
        },
        {
          "id": "B18",
          "title": "Le Scuse Audaci",
          "desc":
              "Avvia una conversazione per scusarti di un errore passato, anche se è successo molto tempo fa.",
        },
        {
          "id": "B19",
          "title": "Il Mentore",
          "desc":
              "Offriti di aiutare qualcuno con meno esperienza di te in una determinata abilità.",
        },
        {
          "id": "B20",
          "title": "L'Architetto Sociale",
          "desc":
              "Crea una nuova tradizione sociale o un incontro ricorrente per un gruppo di amici.",
        },
      ],
    },
    'ru': {
      "Seedling": [
        {
          "id": "S1",
          "title": "Первый шаг",
          "desc":
              "Установи зрительный контакт и улыбнись одному человеку сегодня.",
        },
        {
          "id": "S2",
          "title": "Простое приветствие",
          "desc": "Скажи «Доброе утро» или «Привет» соседу.",
        },
        {
          "id": "S3",
          "title": "Благодарность",
          "desc": "Четко и внятно скажи «Спасибо» продавцу.",
        },
        {
          "id": "S4",
          "title": "Наблюдение",
          "desc": "Замети что-то позитивное в незнакомце и улыбнись.",
        },
        {
          "id": "S5",
          "title": "Тихий взмах",
          "desc": "Помаши рукой тому, кого ты узнал издалека.",
        },
        {
          "id": "S6",
          "title": "Придержи дверь",
          "desc": "Придержи дверь для идущего за тобой человека.",
        },
        {
          "id": "S7",
          "title": "Кивок",
          "desc": "Дружелюбно кивни коллеге, когда проходишь мимо.",
        },
        {
          "id": "S8",
          "title": "Зеркало",
          "desc":
              "Потренируй свою «уверенную улыбку» перед зеркалом в течение 1 минуты.",
        },
        {
          "id": "S9",
          "title": "Краткий взгляд",
          "desc":
              "Посмотри на кого-то в течение 2 секунд, затем улыбнись и отведи взгляд.",
        },
        {
          "id": "S10",
          "title": "Тихая похвала",
          "desc":
              "Напиши добрый комментарий под чьим-то постом в социальных сетях.",
        },
        {
          "id": "S11",
          "title": "Общее пространство",
          "desc":
              "Присядь рядом с кем-то в общественном месте и не отводи взгляд сразу же.",
        },
        {
          "id": "S12",
          "title": "Простая вежливость",
          "desc": "Вежливо скажи «Извините», проходя мимо кого-то в коридоре.",
        },
        {
          "id": "S13",
          "title": "Теплое приветствие",
          "desc": "Скажи «Привет» курьеру или доставщику.",
        },
        {
          "id": "S14",
          "title": "Маленький жест",
          "desc":
              "Помаши рукой ребенку или домашнему животному (с разрешения хозяина).",
        },
        {
          "id": "S15",
          "title": "Мягкая улыбка",
          "desc": "Улыбнись трем разным людям сегодня.",
        },
        {
          "id": "S16",
          "title": "Вызов зрительного контакта",
          "desc":
              "Удерживай зрительный контакт с кассиром, пока он не отведет взгляд первым.",
        },
        {
          "id": "S17",
          "title": "Спокойное дыхание",
          "desc":
              "Сделай 3 глубоких вдоха перед тем, как войти в людное место сегодня.",
        },
        {
          "id": "S18",
          "title": "Присутствие",
          "desc":
              "Побудь в многолюдном месте 5 минут, ни разу не взглянув в телефон.",
        },
        {
          "id": "S19",
          "title": "Случайный кивок",
          "desc": "Кивни незнакомцу, который встретился с тобой взглядом.",
        },
        {
          "id": "S20",
          "title": "Приятное прощание",
          "desc": "Скажи «Хорошего дня» кому-нибудь на выходе из магазина.",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "Комплимент",
          "desc": "Сделай искренний комплимент коллеге или однокласснику.",
        },
        {
          "id": "SP2",
          "title": "Вопрос",
          "desc": "Спроси у незнакомца время или дорогу.",
        },
        {
          "id": "SP3",
          "title": "Небольшой разговор",
          "desc":
              "Спроси кого-нибудь «Как проходит твой день?» и внимательно выслушай ответ.",
        },
        {
          "id": "SP4",
          "title": "Просьба",
          "desc":
              "Попроси сотрудника магазина помочь найти определенный товар.",
        },
        {
          "id": "SP5",
          "title": "Заказ",
          "desc":
              "Закажи напиток или еду и поинтересуйся у персонала, как их дела.",
        },
        {
          "id": "SP6",
          "title": "Знакомство",
          "desc": "Представься кому-то новому в твоем окружении.",
        },
        {
          "id": "SP7",
          "title": "Разговор о погоде",
          "desc":
              "Упомяни погоду в разговоре с кем-то во время ожидания в очереди.",
        },
        {
          "id": "SP8",
          "title": "Простой расспрос",
          "desc": "Спроси у коллеги «Как прошли выходные?»",
        },
        {
          "id": "SP9",
          "title": "Предложение помощи",
          "desc":
              "Спроси «Тебе помочь с этим?», если видишь, что кто-то справляется с трудом.",
        },
        {
          "id": "SP10",
          "title": "Мнение",
          "desc":
              "Спроси друга «Что думаешь об этом?» по поводу какого-то небольшого предмета.",
        },
        {
          "id": "SP11",
          "title": "Уточнение",
          "desc":
              "Уточни деталь у незнакомца (например: «Это правильная очередь?»).",
        },
        {
          "id": "SP12",
          "title": "Окружающая обстановка",
          "desc":
              "Сделай небольшое замечание об обстановке (например: «Здесь действительно многолюдно»).",
        },
        {
          "id": "SP13",
          "title": "Маленькая услуга",
          "desc":
              "Попроси кого-нибудь передать тебе что-то (например, салфетку) за столом.",
        },
        {
          "id": "SP14",
          "title": "Теплый отзыв",
          "desc": "Скажи официанту перед уходом, что еда была великолепной.",
        },
        {
          "id": "SP15",
          "title": "Дружеская проверка",
          "desc":
              "Отправь сообщение «Как дела?» человеку, с которым ты не общался уже месяц.",
        },
        {
          "id": "SP16",
          "title": "Открытый вопрос",
          "desc":
              "Спроси кого-нибудь « Какое твое самое любимое место в этом городе?»",
        },
        {
          "id": "SP17",
          "title": "Минимальный риск",
          "desc":
              "Спроси у незнакомца, знает ли он, где находится ближайший туалет.",
        },
        {
          "id": "SP18",
          "title": "Похвала вещи",
          "desc": "Скажи кому-то, что тебе нравятся его обувь/сумка/аксессуар.",
        },
        {
          "id": "SP19",
          "title": "Вежливая пауза",
          "desc":
              "Подожди, пока человек полностью закончит говорить, прежде чем отвечать ему.",
        },
        {
          "id": "SP20",
          "title": "Дружеский жест",
          "desc":
              "Помаши рукой и скажи «Пока» человеку, с которым у тебя только что было короткое общение.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "Искатель мнений",
          "desc": "Спроси чье-то мнение о книге, фильме или песне.",
        },
        {
          "id": "L2",
          "title": "Детализация",
          "desc":
              "Задай уточняющий вопрос после того, как кто-то рассказал тебе что-то о себе.",
        },
        {
          "id": "L3",
          "title": "Рекомендация",
          "desc":
              "Попроси незнакомца порекомендовать хорошее место, где можно поесть поблизости.",
        },
        {
          "id": "L4",
          "title": "Связующая нить",
          "desc":
              "Найди общий интерес с кем-то и поговори об этом в течение 2 минут.",
        },
        {
          "id": "L5",
          "title": "Рука помощи",
          "desc":
              "Предложи помочь кому-то с небольшой задачей (например, понести сумку).",
        },
        {
          "id": "L6",
          "title": "Социальное наблюдение",
          "desc":
              "Начни разговор, основываясь на том, что происходит вокруг вас обоих.",
        },
        {
          "id": "L7",
          "title": "Открытый вопрос",
          "desc": "Спроси кого-нибудь «Как ты попал в эту сферу работы?»",
        },
        {
          "id": "L8",
          "title": "Активный слушатель",
          "desc":
              "Слушай кого-то в течение 3 минут, не прерывая, а затем кратко перескажи то, что он сказал.",
        },
        {
          "id": "L9",
          "title": "Общий смех",
          "desc":
              "Расскажи короткую смешную историю или анекдот небольшой группе.",
        },
        {
          "id": "L10",
          "title": "Любопытство",
          "desc":
              "Спроси кого-нибудь, откуда он родом и что ему нравится в том месте.",
        },
        {
          "id": "L11",
          "title": "Искренний интерес",
          "desc": "Спроси коллегу о его хобби вне работы.",
        },
        {
          "id": "L12",
          "title": "Мягкий совет",
          "desc":
              "Дай кому-то полезный совет в том, в чем ты хорошо разбираешься.",
        },
        {
          "id": "L13",
          "title": "Групповой кивок",
          "desc": "Согласись с мнением кого-то в дискуссии небольшой группы.",
        },
        {
          "id": "L14",
          "title": "Неформальное приглашение",
          "desc": "Спроси кого-нибудь «Не хочешь пообедать с нами?»",
        },
        {
          "id": "L15",
          "title": "Честная рефлексия",
          "desc":
              "Скажи кому-то «Я действительно оценил, когда ты сделал X» и объясни почему.",
        },
        {
          "id": "L16",
          "title": "Зона любопытства",
          "desc":
              "Спроси кого-нибудь «Мне всегда было интересно, как на самом деле работает X?»",
        },
        {
          "id": "L17",
          "title": "Лидер малой группы",
          "desc":
              "Задай вопрос, который потребует ответа 2 или 3 человек в группе.",
        },
        {
          "id": "L18",
          "title": "Искренний комплимент",
          "desc":
              "Сделай кому-то комплимент за черту характера (например: «Ты отличный слушатель» Balkans).",
        },
        {
          "id": "L19",
          "title": "Общий опыт",
          "desc": "Скажи «Я тоже бывал в такой ситуации» во время разговора.",
        },
        {
          "id": "L20",
          "title": "Значимая пауза",
          "desc":
              "Позволь тишине возникнуть в разговоре, не торопясь заполнить её словами.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "Смелое начало",
          "desc": "Заведи разговор с кем-то, кого ты не очень хорошо знаещь.",
        },
        {
          "id": "ST2",
          "title": "Честное откровение",
          "desc": "Поделись небольшой личной историей или мнением в группе.",
        },
        {
          "id": "ST3",
          "title": "Дискуссия",
          "desc": "Вежливо не согласись с чьим-то мнением и объясни почему.",
        },
        {
          "id": "ST4",
          "title": "Вход в группу",
          "desc":
              "Присоединись к групповому разговору и добавь одну обдуманную фразу.",
        },
        {
          "id": "ST5",
          "title": "Ведение темы",
          "desc": "Подними новую тему для разговора в социальной группе.",
        },
        {
          "id": "ST6",
          "title": "Публичный вопрос",
          "desc": "Задай вопрос на общем собрании или в учебной аудитории.",
        },
        {
          "id": "ST7",
          "title": "Смелая просьба",
          "desc":
              "Спроси незнакомца, можешь ли ты сесть рядом с ним в кафе или парке.",
        },
        {
          "id": "ST8",
          "title": "Разговорный мост",
          "desc":
              "Познакомь двух людей, которые не знают друг друга, и найди для них что-то общее.",
        },
        {
          "id": "ST9",
          "title": "Уверенное требование",
          "desc":
              "Вежливо попроси кого-то пересесть или перестать делать то, что тебя беспокоит.",
        },
        {
          "id": "ST10",
          "title": "Рассказчик",
          "desc":
              "Возьми на себя инициативу в рассказе истории группе из 3 или более человек.",
        },
        {
          "id": "ST11",
          "title": "Открытый вызов",
          "desc":
              "Оспорь общее мнение в группе дружелюбным и уважительным образом.",
        },
        {
          "id": "ST12",
          "title": "Социальная инициатива",
          "desc":
              "Будь первым, кто скажет «Привет» всем при входе в помещение.",
        },
        {
          "id": "ST13",
          "title": "Эмпатичное слушание",
          "desc":
              "Выслушай кого-то, кто изливает душу, и дай поддерживающий ответ.",
        },
        {
          "id": "ST14",
          "title": "Публичная презентация",
          "desc":
              "Поговори 1-2 минуты о теме, которую ты любишь, на общественном собрании.",
        },
        {
          "id": "ST15",
          "title": "Признание уязвимости",
          "desc":
              "Признайся группе, что ты волновался из-за чего-то, и посмейтесь над этим вместе.",
        },
        {
          "id": "ST16",
          "title": "Установка границ",
          "desc":
              "Вежливо отклони приглашение на мероприятие, которое ты не хочешь посещать, без лишних объяснений.",
        },
        {
          "id": "ST17",
          "title": "Активный медиатор",
          "desc": "Помоги двум людям найти компромисс в разногласии.",
        },
        {
          "id": "ST18",
          "title": "Публичный комплимент",
          "desc": "Публично похвали чьи-то усилия или достижения в группе.",
        },
        {
          "id": "ST19",
          "title": "Прямой подход",
          "desc":
              "Попроси кого-то напрямую об услуге или совете, который тебе необходим.",
        },
        {
          "id": "ST20",
          "title": "Поворот разговора",
          "desc": "Плавно переведи разговор со скучной темы на интересную.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "Подарок",
          "desc":
              "Угости кого-нибудь или сделай маленький подарок и скажи: «Я подумал, тебе это понравится».",
        },
        {
          "id": "B2",
          "title": "Смелое лидерство",
          "desc":
              "Предложи план или место для посещения небольшой группе людей.",
        },
        {
          "id": "B3",
          "title": "Признательность",
          "desc":
              "Скажи человеку конкретно, почему ты ценишь его присутствие в твоей жизни.",
        },
        {
          "id": "B4",
          "title": "Социальный организатор",
          "desc":
              "Организуй небольшую встречу или совместный кофе для нескольких человек.",
        },
        {
          "id": "B5",
          "title": "Глубокое погружение",
          "desc":
              "Проведи глубокий, значимый разговор с кем-то продолжительностью более 15 минут.",
        },
        {
          "id": "B6",
          "title": "Пик уверенности",
          "desc":
              "Начни разговор с кем-то, кто кажется тебе пугающим или неприступным.",
        },
        {
          "id": "B7",
          "title": "Публичный тост",
          "desc":
              "Произнеси короткий, позитивный тост или похвалу в адрес кого-то в группе.",
        },
        {
          "id": "B8",
          "title": "Установитель границ",
          "desc":
              "Скажи «Нет» на чью-то просьбу твердо, но вежливо, без оправданий.",
        },
        {
          "id": "B9",
          "title": "Прямой запрос",
          "desc":
              "Попроси человека, которым ты восхищаешься, о 10-минутной беседе или наставничестве.",
        },
        {
          "id": "B10",
          "title": "Эмоциональный лидер",
          "desc": "Начни разговор о чувствах или ментальном здоровье с другом.",
        },
        {
          "id": "B11",
          "title": "Социальный посредник",
          "desc":
              "Помоги двум людям разрешить небольшой конфликт посредством спокойного разговора.",
        },
        {
          "id": "B12",
          "title": "Смелый комплимент",
          "desc":
              "Скажи совершенно незнакомому человеку то, чем ты искренне восхищаешься в нем.",
        },
        {
          "id": "B13",
          "title": "Сетевой шаг",
          "desc":
              "Представься профессионалу в своей области и попроси у него совета.",
        },
        {
          "id": "B14",
          "title": "Смелая правда",
          "desc":
              "Скажи кому-то правду, которая дается трудно, но полезна для отношений.",
        },
        {
          "id": "B15",
          "title": "Полный цвет",
          "desc":
              "Организуй небольшое социальное мероприятие и позаботься о том, чтобы каждый гость чувствовал себя желанным.",
        },
        {
          "id": "B16",
          "title": "Публичный оратор",
          "desc":
              "Вызовись добровольцем выступить или провести небольшую часть собрания или мероприятия.",
        },
        {
          "id": "B17",
          "title": "Лидерство через уязвимость",
          "desc":
              "Поделись трудностями, которые ты преодолел, чтобы подбодрить кого-то другого.",
        },
        {
          "id": "B18",
          "title": "Смелое извинение",
          "desc":
              "Начни разговор, чтобы извиниться за прошлую ошибку, даже если это было очень давно.",
        },
        {
          "id": "B19",
          "title": "Наставник",
          "desc":
              "Предложи помочь кому-то, кто менее опытен, чем ты, освоить какой-либо навык.",
        },
        {
          "id": "B20",
          "title": "Социальный архитектор",
          "desc":
              "Создай новую социальную традицию или регулярную встречу для группы друзей.",
        },
      ],
    },
    'tr': {
      "Seedling": [
        {
          "id": "S1",
          "title": "İlk Adım",
          "desc": "Bugün bir kişiyle göz teması kur ve ona gülümse.",
        },
        {
          "id": "S2",
          "title": "Basit Bir Merhaba",
          "desc": "Bir komşuna 'Günaydın' veya 'Merhaba' de.",
        },
        {
          "id": "S3",
          "title": "Teşekkür",
          "desc":
              "Bir esnafa net ve anlaşılır bir şekilde 'Teşekkür ederim' de.",
        },
        {
          "id": "S4",
          "title": "Gözlem",
          "desc": "Yabancı birinde olumlu bir şey fark et ve gülümse.",
        },
        {
          "id": "S5",
          "title": "Sessiz El Sallama",
          "desc": "Uzaktan tanıdığın birine sakince el salla.",
        },
        {
          "id": "S6",
          "title": "Kapıyı Tutmak",
          "desc": "Arkandan gelen biri için kapıyı açık tut.",
        },
        {
          "id": "S7",
          "title": "Selam Dosyası",
          "desc":
              "Mesaiden yanından geçerken bir iş arkadaşına dostça başınla selam ver.",
        },
        {
          "id": "S8",
          "title": "Ayna Çalışması",
          "desc":
              "Ayna karşısında 1 dakika boyunca 'kendinden emin gülümseme' pratiği yap.",
        },
        {
          "id": "S9",
          "title": "Kısa Bir Bakış",
          "desc": "Birine 2 saniye bak, ardından gülümse ve bakışını kaçır.",
        },
        {
          "id": "S10",
          "title": "Sessiz Övgü",
          "desc":
              "Sosyal medyada birinin gönderisinin altına hoş bir yorum yaz.",
        },
        {
          "id": "S11",
          "title": "Alanı Paylaşmak",
          "desc":
              "Kamusal bir alanda birinin yanına otur ve hemen bakışlarını başka yere çevirme.",
        },
        {
          "id": "S12",
          "title": "Basit Bir Nezaket",
          "desc":
              "Koridorda birinin yanından geçerken kibarca 'Affedersiniz' de.",
        },
        {
          "id": "S13",
          "title": "Sıcak Bir Karşılama",
          "desc": "Bir kuryeye veya dağıtım görevlisine 'Merhaba' de.",
        },
        {
          "id": "S14",
          "title": "Küçük Bir Selam",
          "desc":
              "Bir çocuğa veya (sahibinin izniyle) bir evcil hayvana el salla.",
        },
        {
          "id": "S15",
          "title": "Yumuşak Bir Tebessüm",
          "desc": "Bugün üç farklı kişiye gülümse.",
        },
        {
          "id": "S16",
          "title": "Göz Teması Meydan Okuması",
          "desc":
              "Kasiyer ilk önce bakışını kaçırana kadar onunla göz temasını koru.",
        },
        {
          "id": "S17",
          "title": "Sakin Nefes",
          "desc": "Bugün sosyal bir alana girmeden önce 3 derin nefes al.",
        },
        {
          "id": "S18",
          "title": "Anda Kalmak",
          "desc":
              "Kalabalık bir alanda telefonuna hiç bakmadan 5 dakika boyunca dur.",
        },
        {
          "id": "S19",
          "title": "Doğal Bir Baş Selamı",
          "desc":
              "Seninle göz teması kuran bir yabancıya başınla hafifçe selam ver.",
        },
        {
          "id": "S20",
          "title": "Kibarca Uğurlama",
          "desc": "Bir mağazadan çıkarken birine 'İyi günler' de.",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "İltifat",
          "desc":
              "Bir iş arkadaşına veya sınıf arkadaşına içten bir iltifat et.",
        },
        {
          "id": "SP2",
          "title": "Soru Sormak",
          "desc": "Yabancı birine saati veya yönü sor.",
        },
        {
          "id": "SP3",
          "title": "Ayaküstü Sohbet",
          "desc": "Birine 'Gününüz nasıl geçiyor?' diye sor ve cevabını dinle.",
        },
        {
          "id": "SP4",
          "title": "Yardım İsteme",
          "desc":
              "Bir mağaza çalışanından belirli bir ürünü bulmana yardım etmesini iste.",
        },
        {
          "id": "SP5",
          "title": "Sipariş Esnası",
          "desc":
              "Bir içecek veya yemek sipariş et ve personele nasıl olduklarını sor.",
        },
        {
          "id": "SP6",
          "title": "Tanışma",
          "desc": "Bulunduğun çevredeki yeni birine kendini tanıt.",
        },
        {
          "id": "SP7",
          "title": "Hava Durumu Sohbeti",
          "desc": "Sırada beklerken birine hava durumundan bahset.",
        },
        {
          "id": "SP8",
          "title": "Basit Bir Merak",
          "desc": "Bir iş arkadaşına 'Hafta sonu ne yaptın?' diye sor.",
        },
        {
          "id": "SP9",
          "title": "Yardım Teklifi",
          "desc":
              "Zorlandığını gördüğün birine 'Bununla ilgili bir yardıma ihtiyacın var mı?' diye sor.",
        },
        {
          "id": "SP10",
          "title": "Fikir Almak",
          "desc":
              "Küçük bir nesne hakkında bir arkadaşına 'Bunun hakkında ne düşünüyorsun?' diye sor.",
        },
        {
          "id": "SP11",
          "title": "Teyit Etmek",
          "desc":
              "Bir yabancıyla bir detayı teyit et (Örn: 'Doğru sıra burası mı?').",
        },
        {
          "id": "SP12",
          "title": "Ortam Hakkında",
          "desc":
              "Çevre hakkında küçük bir yorum yap (Örn: 'Burası gerçekten çok kalabalıkmış').",
        },
        {
          "id": "SP13",
          "title": "Küçük Bir Rica",
          "desc":
              "Masada birinden sana bir şeyi (peçete gibi) uzatmasını rica et.",
        },
        {
          "id": "SP14",
          "title": "İçten Bir Bildirim",
          "desc": "Ayrılmadan önce bir garsona yemeğin harika olduğunu söyle.",
        },
        {
          "id": "SP15",
          "title": "Gündelik Hatır Sorma",
          "desc":
              "Bir aydır konuşmadığın birine 'Nasıl gidiyor, nasılsın?' mesajı gönder.",
        },
        {
          "id": "SP16",
          "title": "Açık Uçlu Soru",
          "desc":
              "Birine 'Bu şehirde en sevdiğin, gezilecek yer neresi?' diye sor.",
        },
        {
          "id": "SP17",
          "title": "Minimum Risk",
          "desc":
              "Yabancı birine en yakın lavabonun nerede olduğunu bilip bilmediğini sor.",
        },
        {
          "id": "SP18",
          "title": "Eşya Övgüsü",
          "desc":
              "Birine ayakkabılarını/çantasını/aksesuarını beğendiğini söyle.",
        },
        {
          "id": "SP19",
          "title": "Kibarca Bekleme",
          "desc":
              "Biriyle konuşurken ona cevap vermeden önce sözünü tamamen bitirmesini bekle.",
        },
        {
          "id": "SP20",
          "title": "Dostça Uğurlama",
          "desc":
              "Az önce kısa bir etkileşimde bulunduğun birine el salla ve 'Görüşürüz' de.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "Fikir Arayıcı",
          "desc": "Birine bir kitap, film veya şarkı hakkındaki fikrini sor.",
        },
        {
          "id": "L2",
          "title": "Detaylandırma",
          "desc":
              "Biri sana kendisi hakkında bir şey anlattıktan sonra devam niteliğinde bir soru sor.",
        },
        {
          "id": "L3",
          "title": "Tavsiye İsteme",
          "desc":
              "Yabancı birine yakınlarda yemek yenebilecek iyi bir yer tavsiyesi sor.",
        },
        {
          "id": "L4",
          "title": "Ortak Nokta",
          "desc":
              "Biriyle ortak bir ilgi alanı bul ve bunun hakkında 2 dakika konuş.",
        },
        {
          "id": "L5",
          "title": "Yardımcı El",
          "desc":
              "Birine küçük bir görevde (çanta taşımak gibi) yardım etmeyi teklif et.",
        },
        {
          "id": "L6",
          "title": "Sosyal Gözlem",
          "desc":
              "İkinizin de etrafında gerçekleşen bir şeye dayanarak bir konuşma başlat.",
        },
        {
          "id": "L7",
          "title": "Açık Soru",
          "desc": "Birine 'Bu sektöre veya işe nasıl girdin?' diye sor.",
        },
        {
          "id": "L8",
          "title": "Aktif Dinleyici",
          "desc":
              "Birini sözünü kesmeden 3 dakika dinle, ardından ne söylediğini kısaca özetle.",
        },
        {
          "id": "L9",
          "title": "Ortak Kahkaha",
          "desc":
              "Küçük bir gruba kısa, komik bir hikaye veya bir fıkra anlat.",
        },
        {
          "id": "L10",
          "title": "Merak Duygusu",
          "desc":
              "Birine nereli olduğunu ve o yerin en çok neyini sevdiğini sor.",
        },
        {
          "id": "L11",
          "title": "İçten İlgi",
          "desc": "Bir iş arkadaşına iş dışındaki hobilerini sor.",
        },
        {
          "id": "L12",
          "title": "Küçük Bir İpucu",
          "desc": "İyi olduğun bir konuda birine faydalı bir ipucu ver.",
        },
        {
          "id": "L13",
          "title": "Grup İçi Onay",
          "desc":
              "Küçük bir grup tartışmasında birinin fikrine katıldığını belirtir şekilde başını salla.",
        },
        {
          "id": "L14",
          "title": "Gündelik Davet",
          "desc":
              "Birine 'Öğle yemeğinde bize katılmak ister misin?' diye sor.",
        },
        {
          "id": "L15",
          "title": "Dürüst Geri Bildirim",
          "desc":
              "Birine 'Şunu (X) yaptığında gerçekten çok memnun kalmıştım' de ve nedenini açıkla.",
        },
        {
          "id": "L16",
          "title": "Merak Boşluğu",
          "desc":
              "Birine 'Her zaman merak etmişimdir, şu (X) aslında nasıl çalışıyor?' diye sor.",
        },
        {
          "id": "L17",
          "title": "Grubu Yönlendirme",
          "desc":
              "Gruptaki 2 veya 3 kişinin yanıt vermesini gerektirecek bir soru sor.",
        },
        {
          "id": "L18",
          "title": "Karakter Övgüsü",
          "desc":
              "Birini bir kişilik özelliği üzerinden öv (Örn: 'Harika bir dinleyicisin').",
        },
        {
          "id": "L19",
          "title": "Ortak Deneyim",
          "desc": "Bir sohbet sırasında 'Ben de o durumda bulunmuştum' de.",
        },
        {
          "id": "L20",
          "title": "Bilinçli Duraksama",
          "desc":
              "Bir sohbette lafı hemen doldurmaya çalışmadan sessizliğin oluşmasına izin ver.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "Cesur Bir Başlangıç",
          "desc": "Çok iyi tanımadığın biriyle bir sohbet başlat.",
        },
        {
          "id": "ST2",
          "title": "Dürüst Paylaşım",
          "desc": "Grup ortamında küçük bir kişisel hikaye veya fikir paylaş.",
        },
        {
          "id": "ST3",
          "title": "Fikir Ayrılığı",
          "desc":
              "Birinin fikrine kibarca katılmadığını belirt ve nedenini açıkla.",
        },
        {
          "id": "ST4",
          "title": "Gruba Giriş",
          "desc":
              "Grup sohbetine dahil ol ve düşünceli bir cümleyle katkıda bulun.",
        },
        {
          "id": "ST5",
          "title": "Konu Açmak",
          "desc": "Sosyal bir grupta yeni bir sohbet konusu ortaya at.",
        },
        {
          "id": "ST6",
          "title": "Topluluk Önünde Soru",
          "desc":
              "Halka açık bir toplantıda veya sınıf ortamında bir soru sor.",
        },
        {
          "id": "ST7",
          "title": "Cesur Rica",
          "desc":
              "Bir yabancıya bir kafede veya parkta yanına oturup oturamayacağını sor.",
        },
        {
          "id": "ST8",
          "title": "Sohbet Köprüsü",
          "desc":
              "Birbirini tanımayan iki insanı tanıştır ve ortak bir nokta bul.",
        },
        {
          "id": "ST9",
          "title": "Kendini Savunma",
          "desc":
              "Seni rahatsız eden bir şey yapan birinden kibarca yer değiştirmesini veya bunu durdurmasını iste.",
        },
        {
          "id": "ST10",
          "title": "Hikaye Anlatıcı",
          "desc":
              "3 veya daha fazla kişiden oluşan bir gruba hikaye anlatma rolünü üstlen.",
        },
        {
          "id": "ST11",
          "title": "Saygılı Sorgulama",
          "desc":
              "Gruptaki yaygın bir görüşe arkadaşça ve saygılı bir şekilde meydan oku.",
        },
        {
          "id": "ST12",
          "title": "Sosyal Girişkenlik",
          "desc": "Bir odaya girerken herkese ilk 'Merhaba' diyen kişi sen ol.",
        },
        {
          "id": "ST13",
          "title": "Empatik Dinleme",
          "desc": "İçini döken birini dinle ve destekleyici bir yanıt ver.",
        },
        {
          "id": "ST14",
          "title": "Kısa Sunum",
          "desc":
              "Sosyal bir toplantıda sevdiğin bir konu hakkında 1-2 dakika konuş.",
        },
        {
          "id": "ST15",
          "title": "Kırılganlığı Paylaşmak",
          "desc":
              "Bir konuda gergin olduğunu bir gruba itiraf et ve buna birlikte gülün.",
        },
        {
          "id": "ST16",
          "title": "Sınır Koymak",
          "desc":
              "Katılmak istemediğin bir daveti, aşırı açıklama yapmadan kibarca reddet.",
        },
        {
          "id": "ST17",
          "title": "Aktif Arabulucu",
          "desc":
              "Bir anlaşmazlıkta iki kişinin orta yolu bulmasına yardımcı ol.",
        },
        {
          "id": "ST18",
          "title": "Topluluk Önünde Övgü",
          "desc": "Bir grupta birinin çabasını veya başarısını açıkça öv.",
        },
        {
          "id": "ST19",
          "title": "Doğrudan Yaklaşım",
          "desc":
              "İhtiyacın olan bir iyiliği veya tavsiyeyi birinden doğrudan iste.",
        },
        {
          "id": "ST20",
          "title": "Konuyu Eksenleme",
          "desc":
              "Bir sohbeti sıkıcı bir konudan ilginç bir konuya pürüzsüzce kaydır.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "Hediye",
          "desc":
              "Birine küçük bir ikram ver ve 'Bunu seveceğini düşündüm' de.",
        },
        {
          "id": "B2",
          "title": "Cesur Liderlik",
          "desc":
              "Küçük bir insan grubuna bir plan veya ziyaret edilecek bir yer öner.",
        },
        {
          "id": "B3",
          "title": "Minnettarlık",
          "desc":
              "Birine hayatında olduğu için neden minnettar olduğunu özellikle belirt.",
        },
        {
          "id": "B4",
          "title": "Sosyal Ev Sahibi",
          "desc":
              "Birkaç kişi için küçük bir buluşma veya bir kahve randevusu organize et.",
        },
        {
          "id": "B5",
          "title": "Derin Dalış",
          "desc":
              "Biriyle 15 dakikadan uzun süren derin, anlamlı bir konuşma yap.",
        },
        {
          "id": "B6",
          "title": "Güven Zirvesi",
          "desc": "Gözünü korkuttuğunu düşündüğün biriyle bir sohbet başlat.",
        },
        {
          "id": "B7",
          "title": "Topluluk Önünde Konuşma",
          "desc":
              "Bir grupta biri için kısa, olumlu bir konuşma veya teşekkür jesti yap.",
        },
        {
          "id": "B8",
          "title": "Sınır Çizen",
          "desc":
              "Bir isteğe aşırı açıklama yapmadan, kararlı ama nazik bir şekilde 'Hayır' de.",
        },
        {
          "id": "B9",
          "title": "Doğrudan İstek",
          "desc":
              "Hayran olduğun birinden 10 dakikalık bir sohbet veya akıl hocalığı talep et.",
        },
        {
          "id": "B10",
          "title": "Duygusal Liderlik",
          "desc":
              "Bir arkadaşınla duygular veya ruh sağlığı hakkında bir sohbet başlat.",
        },
        {
          "id": "B11",
          "title": "Sosyal Arabulucu",
          "desc":
              "İki kişinin sakin bir konuşma yoluyla küçük bir çatışmayı çözmesine yardımcı ol.",
        },
        {
          "id": "B12",
          "title": "Cesur İltifat",
          "desc":
              "Tamamen yabancı birine onda gerçekten takdir ettiğin bir şeyi söyle.",
        },
        {
          "id": "B13",
          "title": "Ağ Kurma Adımı",
          "desc": "Alanındaki bir profesyonele kendini tanıt ve tavsiye iste.",
        },
        {
          "id": "B14",
          "title": "Cesur Gerçek",
          "desc": "Birine ilişki için zor ama faydalı olan bir gerçeği söyle.",
        },
        {
          "id": "B15",
          "title": "Tam Çiçek Açma",
          "desc":
              "Küçük bir sosyal etkinlik düzenle ve her konuğun hoş karşılandığından emin ol.",
        },
        {
          "id": "B16",
          "title": "Kamu Konuşmacısı",
          "desc":
              "Bir toplantının veya etkinliğin küçük bir bölümünü konuşmak veya yönetmek için gönüllü ol.",
        },
        {
          "id": "B17",
          "title": "Kırılgan Liderlik",
          "desc":
              "Başka birini cesaretlendirmek için aştığın bir zorluğu onunla paylaş.",
        },
        {
          "id": "B18",
          "title": "Cesur Özür",
          "desc":
              "Çok uzun zaman önce olmuş olsa bile geçmiş bir hata için özür dilemek üzere bir konuşma başlat.",
        },
        {
          "id": "B19",
          "title": "Akıl Hocası",
          "desc":
              "Bir yetenek konusunda senden daha az deneyimli olan birine yardım etmeyi teklif et.",
        },
        {
          "id": "B20",
          "title": "Sosyal Mimar",
          "desc":
              "Bir arkadaş grubu için yeni bir sosyal gelenek veya tekrarlayan bir buluşma oluştur.",
        },
      ],
    },
    'nl': {
      "Seedling": [
        {
          "id": "S1",
          "title": "De Eerste Stap",
          "desc": "Maak vandaag oogcontact en glimlach naar één persoon.",
        },
        {
          "id": "S2",
          "title": "Een Simpel Hallo",
          "desc": "Zeg 'Goedemorgen' of 'Hallo' tegen een buur.",
        },
        {
          "id": "S3",
          "title": "Het Bedankt",
          "desc":
              "Zeg duidelijk 'Dank u wel' of 'Bedankt' tegen een winkelier.",
        },
        {
          "id": "S4",
          "title": "De Observatie",
          "desc": "Merk iets positiefs op aan een vreemde en glimlach.",
        },
        {
          "id": "S5",
          "title": "De Stille Zwaai",
          "desc": "Zwaai naar iemand die je herkent van een afstand.",
        },
        {
          "id": "S6",
          "title": "Deur Openhouden",
          "desc": "Houd de deur open voor iemand achter je.",
        },
        {
          "id": "S7",
          "title": "De Knik",
          "desc":
              "Geef een vriendelijk knikje aan een collega als je die voorbijloopt.",
        },
        {
          "id": "S8",
          "title": "De Spiegel",
          "desc":
              "Oefen je 'zelfverzekerde glimlach' gedurende 1 minuut in de spiegel.",
        },
        {
          "id": "S9",
          "title": "De Korte Blik",
          "desc": "Kijk iemand 2 seconden aan, glimlach en kijk dan weg.",
        },
        {
          "id": "S10",
          "title": "De Stille Compliment",
          "desc": "Schrijf een aardige reactie op iemands social media post.",
        },
        {
          "id": "S11",
          "title": "Ruimte Delen",
          "desc":
              "Ga naast iemand zitten in een openbare ruimte zonder meteen weg te kijken.",
        },
        {
          "id": "S12",
          "title": "De Simpele Erkenning",
          "desc":
              "Zeg beleefd 'Pardon' of 'Sorry' als je iemand passeert in een gang.",
        },
        {
          "id": "S13",
          "title": "De Warme Groet",
          "desc": "Zeg 'Hallo' tegen een bezorger of koerier.",
        },
        {
          "id": "S14",
          "title": "De Kleine Zwaai",
          "desc":
              "Zwaai naar een kind of een huisdier (met toestemming van de eigenaar).",
        },
        {
          "id": "S15",
          "title": "De Zachte Glimlach",
          "desc": "Glimlach vandaag naar drie verschillende mensen.",
        },
        {
          "id": "S16",
          "title": "De Oogcontact Uitdaging",
          "desc":
              "Houd oogcontact met een kassière totdat diegene als eerste wegkijkt.",
        },
        {
          "id": "S17",
          "title": "De Rustige Ademhaling",
          "desc":
              "Haal vandaag 3 keer diep adem voordat je een sociale ruimte betreedt.",
        },
        {
          "id": "S18",
          "title": "De Aanwezigheid",
          "desc":
              "Sta 5 minuten in een drukke ruimte zonder op je telefoon te kijken.",
        },
        {
          "id": "S19",
          "title": "De Toevallige Knik",
          "desc": "Knik naar een vreemde die oogcontact met je maakt.",
        },
        {
          "id": "S20",
          "title": "De Zachte Stem",
          "desc": "Zeg 'Fijne dag nog' tegen iemand als je een winkel verlaat.",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "Het Compliment",
          "desc": "Geef een oprecht compliment aan een collega of klasgenoot.",
        },
        {
          "id": "SP2",
          "title": "De Vraag",
          "desc": "Vraag een vreemde naar de tijd of de weg.",
        },
        {
          "id": "SP3",
          "title": "Small Talk",
          "desc":
              "Vraag iemand 'Hoe gaat je dag?' en luister naar het antwoord.",
        },
        {
          "id": "SP4",
          "title": "Het Verzoek",
          "desc":
              "Vraag een winkelmedewerker om hulp bij het vinden van een specifiek artikel.",
        },
        {
          "id": "SP5",
          "title": "De Bestelling",
          "desc":
              "Bestel een drankje of eten en vraag aan het personeel hoe het met ze gaat.",
        },
        {
          "id": "SP6",
          "title": "De Kennismaking",
          "desc": "Stel jezelf voor aan iemand die nieuw is in jouw omgeving.",
        },
        {
          "id": "SP7",
          "title": "Praten over het Weer",
          "desc":
              "Begin over het weer tegen iemand terwijl je in de rij staat te wachten.",
        },
        {
          "id": "SP8",
          "title": "De Simpele Navraag",
          "desc": "Vraag een collega 'Wat heb je dit weekend gedaan?'",
        },
        {
          "id": "SP9",
          "title": "Het Hulpaanbod",
          "desc":
              "Vraag iemand 'Heb je daar hulp bij nodig?' als diegene er moeizaam uitziet.",
        },
        {
          "id": "SP10",
          "title": "De Mening",
          "desc":
              "Vraag een vriend 'Wat vind je hiervan?' over een klein voorwerp.",
        },
        {
          "id": "SP11",
          "title": "De Bevestiging",
          "desc":
              "Bevestig een detail bij een vreemde (bijv. 'Is dit de juiste rij?').",
        },
        {
          "id": "SP12",
          "title": "De Gedeelde Ruimte",
          "desc":
              "Maak een kleine opmerking over de omgeving (bijv. 'Het is echt heel druk hier').",
        },
        {
          "id": "SP13",
          "title": "De Kleine Gunst",
          "desc":
              "Vraag iemand aan tafel om iets aan te geven (zoals een servet).",
        },
        {
          "id": "SP14",
          "title": "De Warme Feedback",
          "desc":
              "Vertel een ober dat het eten geweldig was voordat je weggaat.",
        },
        {
          "id": "SP15",
          "title": "De Informele Check-in",
          "desc":
              "Stuur een 'Hoe gaat het?'-berichtje naar iemand die je al een maand niet hebt gesproken.",
        },
        {
          "id": "SP16",
          "title": "De Open Vraag",
          "desc":
              "Vraag iemand 'Wat is je favoriete plek om te bezoeken in deze stad?'",
        },
        {
          "id": "SP17",
          "title": "Het Kleinste Risico",
          "desc":
              "Vraag een vreemde of diegene weet waar het dichtstbijzijnde toilet is.",
        },
        {
          "id": "SP18",
          "title": "De Compliment voor een Item",
          "desc":
              "Vertel iemand dat je diens schoenen/tas/accessoire leuk vindt.",
        },
        {
          "id": "SP19",
          "title": "De Beleefde Pauze",
          "desc":
              "Wacht tot iemand helemaal is uitgesproken voordat je reageert.",
        },
        {
          "id": "SP20",
          "title": "De Vriendelijke Zwaai",
          "desc":
              "Zwaai en zeg 'Doei' tegen iemand met wie je net een korte interactie hebt gehad.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "De Meningszoeker",
          "desc":
              "Vraag iemand naar diens mening over een boek, film of liedje.",
        },
        {
          "id": "L2",
          "title": "Het Detail",
          "desc":
              "Stel een vervolgvraag nadat iemand je iets over zichzelf heeft verteld.",
        },
        {
          "id": "L3",
          "title": "De Aanbeveling",
          "desc":
              "Vraag een vreemde om een aanbeveling voor een goede plek om te eten in de buurt.",
        },
        {
          "id": "L4",
          "title": "De Connectie",
          "desc":
              "Vind een gemeenschappelijke interesse met iemand en praat er 2 minuten over.",
        },
        {
          "id": "L5",
          "title": "De Helpende Hand",
          "desc":
              "Bied aan om iemand te helpen met een kleine taak (zoals een tas dragen).",
        },
        {
          "id": "L6",
          "title": "De Sociale Observatie",
          "desc":
              "Begin een gesprek op basis van iets dat er om jullie heen gebeurt.",
        },
        {
          "id": "L7",
          "title": "De Open Vraag",
          "desc": "Vraag iemand 'Hoe ben je in dit soort werk terechtgekomen?'",
        },
        {
          "id": "L8",
          "title": "De Actieve Luisteraar",
          "desc":
              "Luister 3 minuten naar iemand zonder te interrumperen, en vat daarna samen wat diegene zei.",
        },
        {
          "id": "L9",
          "title": "De Gedeelde Lach",
          "desc":
              "Vertel een kort, grappig verhaal of een grap aan een klein groepje.",
        },
        {
          "id": "L10",
          "title": "De Nieuwsgierigheid",
          "desc":
              "Vraag iemand waar diegene vandaan komt en wat diegene leuk vindt aan die plek.",
        },
        {
          "id": "L11",
          "title": "De Oprechte Interesse",
          "desc": "Vraag een collega naar diens hobby's buiten het werk.",
        },
        {
          "id": "L12",
          "title": "Het Subtiele Advies",
          "desc":
              "Geef iemand een handige tip over iets waar jij goed in bent.",
        },
        {
          "id": "L13",
          "title": "De Groepsknik",
          "desc":
              "Wees het eens met iemands punt in een kleine groepsdiscussie.",
        },
        {
          "id": "L14",
          "title": "De Informele Uitnodiging",
          "desc":
              "Vraag iemand 'Zou je het leuk vinden om met ons mee te gaan lunchen?'",
        },
        {
          "id": "L15",
          "title": "De Eerlijke Reflectie",
          "desc":
              "Vertel iemand 'Ik stelde het erg op prijs toen je X deed' en leg uit waarom.",
        },
        {
          "id": "L16",
          "title": "Het Nieuwsgierigheidsgat",
          "desc":
              "Vraag iemand 'Ik heb me altijd afgevraagd, hoe werkt X eigenlijk precies?'",
        },
        {
          "id": "L17",
          "title": "De Kleine Groepsleider",
          "desc":
              "Stel een vraag waar 2 of 3 mensen in een groep op moeten antwoorden.",
        },
        {
          "id": "L18",
          "title": "Het Oprechte Compliment",
          "desc":
              "Geef iemand een compliment over een persoonlijkheidskenmerk (bijv. 'Je bent een geweldige luisteraar').",
        },
        {
          "id": "L19",
          "title": "De Gedeelde Ervaring",
          "desc":
              "Zeg 'Ik heb ook in die situatie gezeten' tijdens een gesprek.",
        },
        {
          "id": "L20",
          "title": "De Betekenisvolle Pauze",
          "desc":
              "Laat een stilte vallen in een gesprek zonder je te haasten om deze op te vullen.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "De Dappere Start",
          "desc": "Begin een gesprek met iemand die je niet goed kent.",
        },
        {
          "id": "ST2",
          "title": "Het Eerlijk Delen",
          "desc":
              "Deel een kort persoonlijk verhaal of mening in een groepssetting.",
        },
        {
          "id": "ST3",
          "title": "Het Debat",
          "desc":
              "Wees het op een beleefde manier oneens met iemands mening en leg uit waarom.",
        },
        {
          "id": "ST4",
          "title": "De Groepsdeelname",
          "desc":
              "Sluit je aan bij een groepsgesprek en draag een doordachte zin bij.",
        },
        {
          "id": "ST5",
          "title": "De Onderwerpleider",
          "desc":
              "Breng een nieuw gespreksonderwerp in binnen een sociale groep.",
        },
        {
          "id": "ST6",
          "title": "De Publieke Vraag",
          "desc":
              "Stel een vraag in een openbare vergadering of in een klaslokaal.",
        },
        {
          "id": "ST7",
          "title": "Het Bold Verzoek",
          "desc":
              "Vraag een vreemde of je naast hem of haar mag zitten in een café of park.",
        },
        {
          "id": "ST8",
          "title": "De Gespreksbrug",
          "desc":
              "Introduceer twee mensen die elkaar niet kennen en vind een gemeenschappelijke factor.",
        },
        {
          "id": "ST9",
          "title": "De Assertieve Behoefte",
          "desc":
              "Vraag iemand beleefd om te verplaatsen of op te houden met iets dat je stoort.",
        },
        {
          "id": "ST10",
          "title": "De Verhalenverteller",
          "desc":
              "Neem de leiding bij het vertellen van een verhaal aan een groep van 3 of meer mensen.",
        },
        {
          "id": "ST11",
          "title": "De Open Uitdaging",
          "desc":
              "Daag een algemene mening in een groep uit op een vriendelijke, respectvolle manier.",
        },
        {
          "id": "ST12",
          "title": "Het Sociale Initiatief",
          "desc":
              "Wees de eerste persoon die iedereen 'Hallo' groet bij het binnenkomen van een kamer.",
        },
        {
          "id": "ST13",
          "title": "Het Empathisch Luisteren",
          "desc":
              "Luister naar iemand die zijn hart lucht en geef een ondersteunende reactie.",
        },
        {
          "id": "ST14",
          "title": "De Publieke Presentatie",
          "desc":
              "Praat 1-2 minuten over een onderwerp waar je van houdt tijdens een sociale bijeenkomst.",
        },
        {
          "id": "ST15",
          "title": "Het Kwetsbaar Delen",
          "desc":
              "Geef voor een groep toe dat je ergens nerveus over was, en lach er samen om.",
        },
        {
          "id": "ST16",
          "title": "De Grens Instellen",
          "desc":
              "Sla een uitnodiging waar je niet naartoe wilt beleefd af zonder overmatig veel uitleg te geven.",
        },
        {
          "id": "ST17",
          "title": "De Actieve Mediator",
          "desc":
              "Help twee mensen een middenweg te vinden bij een meningsverschil.",
        },
        {
          "id": "ST18",
          "title": "Het Publieke Compliment",
          "desc":
              "Prijs iemands inzet of prestatie in het openbaar in een groep.",
        },
        {
          "id": "ST19",
          "title": "De Directe Aanpak",
          "desc":
              "Vraag iemand rechtstreeks om een gunst of een stuk advies dat je nodig hebt.",
        },
        {
          "id": "ST20",
          "title": "Het Gesprekspivot",
          "desc":
              "Laat een gesprek soepel overgaan van een saai onderwerp naar een interessant onderwerp.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "Het Cadeautje",
          "desc":
              "Geef een kleine traktatie of attentie aan iemand en zeg 'Ik dacht dat je dit wel leuk zou vinden'.",
        },
        {
          "id": "B2",
          "title": "De Gewaagde Leiding",
          "desc":
              "Stel een plan of een plek om te bezoeken voor aan een klein groepje mensen.",
        },
        {
          "id": "B3",
          "title": "De Waardering",
          "desc":
              "Vertel iemand specifiek waarom je het waardeert dat diegene in je leven is.",
        },
        {
          "id": "B4",
          "title": "De Sociale Gastheer/Gastvrouw",
          "desc":
              "Organiseer een kleine bijeenkomst of een koffie-afspraak voor een paar mensen.",
        },
        {
          "id": "B5",
          "title": "De Diepe Duik",
          "desc":
              "Voer een diep, betekenisvol gesprek met iemand dat langer dan 15 minuten duurt.",
        },
        {
          "id": "B6",
          "title": "De Piek van Zelfvertrouwen",
          "desc": "Begin een gesprek met iemand die je intimiderend vindt.",
        },
        {
          "id": "B7",
          "title": "De Publieke Toast",
          "desc":
              "Breng een korte, positieve toast uit of geef een shout-out naar iemand in een groep.",
        },
        {
          "id": "B8",
          "title": "De Grenssteller",
          "desc":
              "Zeg stevig maar vriendelijk 'Nee' tegen een verzoek, zonder overmatig veel uitleg te geven.",
        },
        {
          "id": "B9",
          "title": "Het Directe Verzoek",
          "desc":
              "Vraag iemand die je bewondert om een gesprek van 10 minuten of om mentorschap.",
        },
        {
          "id": "B10",
          "title": "De Emotionele Leiding",
          "desc":
              "Begin met een vriend(in) een gesprek over gevoelens of mentale gezondheid.",
        },
        {
          "id": "B11",
          "title": "De Sociale Mediator",
          "desc":
              "Help twee mensen een klein conflict op te lossen door middel van een rustig gesprek.",
        },
        {
          "id": "B12",
          "title": "Het Gewaagde Compliment",
          "desc":
              "Zeg tegen een wildvreemde iets wat je oprecht in diegene bewondert.",
        },
        {
          "id": "B13",
          "title": "De Netwerkstap",
          "desc":
              "Stel jezelf voor aan een professional in jouw vakgebied en vraag om advies.",
        },
        {
          "id": "B14",
          "title": "De Moedige Waarheid",
          "desc":
              "Vertel iemand een waarheid die moeilijk is, maar wel helpend voor de relatie.",
        },
        {
          "id": "B15",
          "title": "De Volle Bloei",
          "desc":
              "Organiseer een klein sociaal evenement en zorg ervoor dat elke gast zich welkom voelt.",
        },
        {
          "id": "B16",
          "title": "De Publieke Spreker",
          "desc":
              "Meld je vrijwillig aan om te spreken of een klein deel van een vergadering of evenement te leiden.",
        },
        {
          "id": "B17",
          "title": "De Kwetsbare Leiding",
          "desc":
              "Deel een worsteling die je hebt overwonnen om iemand anders te bemoedigen.",
        },
        {
          "id": "B18",
          "title": "Het Gewaagde Excuus",
          "desc":
              "Begin een gesprek om je te verontschuldigen voor een fout uit het verleden, zelfs als het lang geleden is.",
        },
        {
          "id": "B19",
          "title": "De Mentor",
          "desc":
              "Bied aan om iemand die minder ervaren is dan jij te helpen met een bepaalde vaardigheid.",
        },
        {
          "id": "B20",
          "title": "De Sociale Architect",
          "desc":
              "Creëer een nieuwe sociale traditie of een terugkerende meetup voor een groep vrienden.",
        },
      ],
    },
    'bn': {
      "Seedling": [
        {
          "id": "S1",
          "title": "প্রথম পদক্ষেপ",
          "desc":
              "আজ অন্তত একজনের চোখের দিকে তাকিয়ে তাকান এবং একটি মৃদু হাসি দিন।",
        },
        {
          "id": "S2",
          "title": "একটি সাধারণ হ্যালো",
          "desc": "কোনো প্রতিবেশীকে 'শুভ সকাল' বা 'হ্যালো' বলুন।",
        },
        {
          "id": "S3",
          "title": "ধন্যবাদ জানানো",
          "desc": "কোনো দোকানদারকে স্পষ্ট ও সুন্দরভাবে 'ধন্যবাদ' বলুন।",
        },
        {
          "id": "S4",
          "title": "পর্যবেক্ষণ",
          "desc":
              "কোনো অপরিচিত মানুষের ভালো কোনো দিক খেয়াল করুন এবং মনে মনে হাসুন।",
        },
        {
          "id": "S5",
          "title": "দূর থেকে হাত নাড়া",
          "desc": "দূর থেকে চেনা কোনো মানুষকে দেখে আলতো করে হাত নাড়ুন।",
        },
        {
          "id": "S6",
          "title": "দরজা ধরে রাখা",
          "desc": "আপনার পেছনে আসা কোনো মানুষের জন্য দরজাটি খুলে ধরে রাখুন।",
        },
        {
          "id": "S7",
          "title": "মাথা নাড়িয়ে শুভেচ্ছা",
          "desc":
              "কোনো সহকর্মীর পাশ দিয়ে যাওয়ার সময় মৃদু মাথা নাড়িয়ে শুভেচ্ছা জানান।",
        },
        {
          "id": "S8",
          "title": "আয়নার সামনে অনুশীলন",
          "desc":
              "আয়নার সামনে দাঁড়িয়ে ১ মিনিট আপনার 'আত্মবিশ্বাসী হাসি' অনুশীলন করুন।",
        },
        {
          "id": "S9",
          "title": "এক পলক তাকানো",
          "desc": "কারও দিকে ২ সেকেন্ড তাকান, তারপর একটু হেসে চোখ সরিয়ে নিন।",
        },
        {
          "id": "S10",
          "title": "সুন্দর মন্তব্য",
          "desc":
              "কারও সোশ্যাল মিডিয়া পোস্টে একটি ইতিবাচক ও সুন্দর মন্তব্য লিখুন।",
        },
        {
          "id": "S11",
          "title": "জায়গা ভাগ করে নেওয়া",
          "desc":
              "জনাকীর্ণ কোনো জায়গায় কারও পাশে বসুন এবং সাথে সাথে চোখ সরিয়ে নেবেন না।",
        },
        {
          "id": "S12",
          "title": "ছোট্ট সৌজন্য",
          "desc":
              "বারান্দা বা করিডোরে কারও পাশ দিয়ে যাওয়ার সময় সুন্দরভাবে 'একটু সরবেন' বা 'দুঃখিত' বলুন।",
        },
        {
          "id": "S13",
          "title": "আন্তরিক শুভেচ্ছা",
          "desc":
              "কোনো ডেলিভারি ম্যান বা কুরিয়ার কর্মীকে দেখে 'হ্যালো' বা 'কেমন আছেন' বলুন।",
        },
        {
          "id": "S14",
          "title": "হাত নাড়ানো",
          "desc":
              "কোনো শিশুকে বা পোষা প্রাণীকে হাত নাড়ুন (মালিকের অনুমতি নিয়ে)।",
        },
        {
          "id": "S15",
          "title": "মৃদু হাসি",
          "desc": "আজ তিনটি ভিন্ন ভিন্ন মানুষের দিকে তাকিয়ে হাসুন।",
        },
        {
          "id": "S16",
          "title": "আই কন্টাক্ট চ্যালেঞ্জ",
          "desc":
              "ক্যাশিয়ারের চোখের দিকে তাকিয়ে থাকুন যতক্ষণ না সে নিজে থেকে আগে চোখ সরিয়ে নেয়।",
        },
        {
          "id": "S17",
          "title": "শান্তভাবে শ্বাস নেওয়া",
          "desc":
              "আজ কোনো সামাজিক বা জনাকীর্ণ জায়গায় প্রবেশের আগে ৩ বার দীর্ঘ শ্বাস নিন।",
        },
        {
          "id": "S18",
          "title": "ফোনের বাইরে থাকা",
          "desc":
              "একটি ভিড় জায়গায় ফোনের দিকে এক পলকও না তাকিয়ে ৫ মিনিট দাঁড়িয়ে থাকুন।",
        },
        {
          "id": "S19",
          "title": "সহজ মাথা নাড়া",
          "desc":
              "আপনার দিকে চোখ পড়া কোনো অপরিচিত মানুষকে দেখে আলতো মাথা নাড়ুন।",
        },
        {
          "id": "S20",
          "title": "বিদায়বেলার শুভেচ্ছা",
          "desc": "কোনো দোকান থেকে বের হওয়ার সময় বলুন, 'আপনার দিনটি শুভ হোক'।",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "প্রশংসা করা",
          "desc":
              "আপনার কোনো সহকর্মী বা সহপাঠীর একটি খাঁটি ও মনখোলা প্রশংসা করুন।",
        },
        {
          "id": "SP2",
          "title": "জিজ্ঞাসা করা",
          "desc": "কোনো অপরিচিত মানুষের কাছে সময় অথবা পথ জানতে চান।",
        },
        {
          "id": "SP3",
          "title": "ছোটোখাটো আলাপ",
          "desc":
              "কাউকে জিজ্ঞেস করুন 'আজকের দিনটি আপনার কেমন কাটছে?' এবং তার উত্তরটি মন দিয়ে শুনুন।",
        },
        {
          "id": "SP4",
          "title": "সাহায্য চাওয়া",
          "desc": "কোনো নির্দিষ্ট জিনিস খুঁজে পেতে শপ কর্মচারীর সাহায্য চান।",
        },
        {
          "id": "SP5",
          "title": "অর্ডার দেওয়ার আলাপ",
          "desc":
              "খাবার বা পানীয়ের অর্ডার দেওয়ার সময় কর্মচারীদের জিজ্ঞেস করুন তারা কেমন আছেন।",
        },
        {
          "id": "SP6",
          "title": "পরিচয় দেওয়া",
          "desc": "আপনার চারপাশের নতুন কোনো মানুষের কাছে নিজের পরিচয় দিন।",
        },
        {
          "id": "SP7",
          "title": "আবহাওয়া নিয়ে আলাপ",
          "desc":
              "লাইনে দাঁড়িয়ে অপেক্ষা করার সময় পাশের মানুষের সাথে আবহাওয়া নিয়ে দু-একটি কথা বলুন।",
        },
        {
          "id": "SP8",
          "title": "সহজ খোঁজখবর",
          "desc": "সহকর্মীকে জিজ্ঞেস করুন, 'সাপ্তাহিক ছুটিতে কী করলেন?'",
        },
        {
          "id": "SP9",
          "title": "সাহায্যের হাত",
          "desc":
              "কাউকে সমস্যায় পড়তে দেখলে জিজ্ঞেস করুন, 'আমি কি আপনাকে একটু সাহায্য করতে পারি?'",
        },
        {
          "id": "SP10",
          "title": "মতামত নেওয়া",
          "desc":
              "কোনো বন্ধুকে ছোট একটি জিনিস দেখিয়ে জিজ্ঞেস করুন, 'তোমার এটি কেমন মনে হচ্ছে?'।",
        },
        {
          "id": "SP11",
          "title": "নিশ্চিত হওয়া",
          "desc":
              "কোনো অপরিচিত মানুষের থেকে একটি তথ্য নিশ্চিত হয়ে নিন (যেমন: 'এটিই কি সঠিক লাইন?')।",
        },
        {
          "id": "SP12",
          "title": "পরিবেশ নিয়ে মন্তব্য",
          "desc":
              "চারপাশের পরিবেশ নিয়ে একটি ছোট মন্তব্য করুন (যেমন: 'আজ এখানে সত্যিই অনেক ভিড়')।",
        },
        {
          "id": "SP13",
          "title": "ছোট্ট অনুরোধ",
          "desc":
              "খাওয়ার টেবিলে কাউকে কোনো জিনিস (যেমন ন্যাপকিন) আপনার দিকে এগিয়ে দিতে বলুন।",
        },
        {
          "id": "SP14",
          "title": "ভালো মন্তব্য",
          "desc":
              "রেস্তোরাঁ থেকে বের হওয়ার আগে ওয়েটারকে জানান যে খাবারটি দারুণ ছিল।",
        },
        {
          "id": "SP15",
          "title": "যোগাযোগ করা",
          "desc":
              "যে মানুষের সাথে এক মাস কথা হয়নি, তাকে হুট করে একটি 'কেমন আছেন?' টেক্সট পাঠান।",
        },
        {
          "id": "SP16",
          "title": "উন্মুক্ত প্রশ্ন",
          "desc":
              "কাউকে জিজ্ঞেস করুন, 'এই শহরের কোন জায়গাটি আপনার সবচেয়ে পছন্দের?'।",
        },
        {
          "id": "SP17",
          "title": "ছোট একটি ঝুঁকি",
          "desc":
              "কোনো অপরিচিত মানুষকে জিজ্ঞেস করুন তারা জানেন কি না সবচেয়ে কাছের ওয়াশরুমটি কোথায়।",
        },
        {
          "id": "SP18",
          "title": "জিনিসের প্রশংসা",
          "desc":
              "কাউকে বলুন যে আপনার তাদের জুতো/ব্যাগ/অ্যাক্সেসরিজ পছন্দ হয়েছে।",
        },
        {
          "id": "SP19",
          "title": "ভদ্র বিরতি",
          "desc":
              "কাউকে উত্তর দেওয়ার আগে তার কথাটি সম্পূর্ণ শেষ হওয়া পর্যন্ত অপেক্ষা করুন।",
        },
        {
          "id": "SP20",
          "title": "বিদায় জানানো",
          "desc":
              "যার সাথে একটু আগেই আপনার সংক্ষিপ্ত কথা হয়েছে, তাকে হাত নেড়ে 'বাই' বলুন।",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "মতামত অন্বেষণ",
          "desc": "কোনো বই, সিনেমা বা গান সম্পর্কে কারও মতামত জানতে চান।",
        },
        {
          "id": "L2",
          "title": "পরবর্তী প্রশ্ন",
          "desc":
              "কেউ নিজের সম্পর্কে কিছু বলার পর সেই কথা ধরে আরেকটি প্রাসঙ্গিক প্রশ্ন করুন।",
        },
        {
          "id": "L3",
          "title": "পরামর্শ চাওয়া",
          "desc":
              "কোনো অপরিচিত মানুষের কাছে আশেপাশের ভালো কোনো খাবারের জায়গার সন্ধান চান।",
        },
        {
          "id": "L4",
          "title": "মিল খুঁজে পাওয়া",
          "desc":
              "কারও সাথে একটি সাধারণ আগ্রহের বিষয় খুঁজে বের করুন এবং তা নিয়ে ২ মিনিট কথা বলুন।",
        },
        {
          "id": "L5",
          "title": "সহায়তার হাত",
          "desc":
              "কাউকে ছোট কোনো কাজে সাহায্য করার প্রস্তাব দিন (যেমন ব্যাগ বহন করা)।",
        },
        {
          "id": "L6",
          "title": "চারপাশের পর্যবেক্ষণ",
          "desc":
              "আপনাদের দুইজনের সামনেই ঘটছে এমন কোনো ঘটনাকে কেন্দ্র করে কথা বলা শুরু করুন।",
        },
        {
          "id": "L7",
          "title": "উন্মুক্ত প্রশ্ন",
          "desc": "কাউকে জিজ্ঞেস করুন, 'আপনি কীভাবে এই পেশায় এলেন?'",
        },
        {
          "id": "L8",
          "title": "মনোযোগী শ্রোতা",
          "desc":
              "মাঝখানে কথা না কেটে ৩ মিনিট কারও কথা শুনুন, তারপর তিনি যা বললেন তার সংক্ষিপ্ত রূপ বলুন।",
        },
        {
          "id": "L9",
          "title": "একত্রে হাসা",
          "desc":
              "একটি ছোট দলকে একটি সংক্ষিপ্ত, মজার গল্প বা একটি কৌতুক শোনান।",
        },
        {
          "id": "L10",
          "title": "কৌতূহল প্রকাশ",
          "desc":
              "কাউকে জিজ্ঞেস করুন তাদের হোমটাউন কোথায় এবং সেই জায়গার কোন জিনিসটি তাদের ভালো লাগে।",
        },
        {
          "id": "L11",
          "title": "আন্তরিক আগ্রহ",
          "desc": "সহকর্মীকে কাজের বাইরে তার শখ ও ভালো লাগা নিয়ে প্রশ্ন করুন।",
        },
        {
          "id": "L12",
          "title": "সহজ পরামর্শ",
          "desc":
              "আপনি ভালো পারেন এমন একটি বিষয়ে অন্য কাউকে দরকারি কোনো টিপস দিন।",
        },
        {
          "id": "L13",
          "title": "গ্রুপে সম্মতি",
          "desc":
              "ছোট কোনো গ্রুপ আলোচনায় মাথা নেড়ে কারও যুক্তির সাথে একমত প্রকাশ করুন।",
        },
        {
          "id": "L14",
          "title": "সহজ আমন্ত্রণ",
          "desc":
              "কাউকে জিজ্ঞেস করুন, 'আপনি কি দুপুরের খাবারে আমাদের সাথে যোগ দিতে চান?'",
        },
        {
          "id": "L15",
          "title": "কৃতজ্ঞতা প্রকাশ",
          "desc":
              "কাউকে বলুন, 'আপনি যখন এক্স (X) করেছিলেন তখন আমার খুব ভালো লেগেছিল' এবং কারণ বুঝিয়ে বলুন।",
        },
        {
          "id": "L16",
          "title": "জানার আগ্রহ",
          "desc":
              "কাউকে জিজ্ঞেস করুন, 'আমি সবসময় ভাবি, এক্স (X) আসলে কীভাবে কাজ করে?'",
        },
        {
          "id": "L17",
          "title": "ছোট দলের নেতৃত্ব",
          "desc":
              "এমন একটি প্রশ্ন জিজ্ঞাসা করুন যার উত্তর দিতে একটি দলের ২ বা ৩ জন লোকের প্রয়োজন হয়।",
        },
        {
          "id": "L18",
          "title": "আন্তরিক প্রশংসা",
          "desc":
              "কারও ব্যক্তিত্বের কোনো বৈশিষ্ট্যের প্রশংসা করুন (যেমন, 'আপনি খুব ভালো শ্রোতা')।",
        },
        {
          "id": "L19",
          "title": "ভাগ করা অভিজ্ঞতা",
          "desc": "কথোপকথনের সময় বলুন 'আমিও এই পরিস্থিতিতে পড়েছিলাম'।",
        },
        {
          "id": "L20",
          "title": "তাৎপর্যপূর্ণ বিরতি",
          "desc":
              "কথোপকথনে কোনো নীরবতার মুহূর্ত তৈরি হলে তা তাড়াহুড়ো করে পূরণ না করে স্বাভাবিক থাকতে দিন।",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "সাহসী সূচনা",
          "desc":
              "এমন কারও সাথে কথোপকথন শুরু করুন যাকে আপনি খুব ভালো করে চেনেন না।",
        },
        {
          "id": "ST2",
          "title": "সততার সাথে ভাগ করা",
          "desc":
              "একটি দলগত পরিবেশে নিজের একটি ছোট ব্যক্তিগত গল্প বা মতামত শেয়ার করুন।",
        },
        {
          "id": "ST3",
          "title": "যুক্তিতর্ক",
          "desc":
              "কারও মতামতের সাথে ভদ্রভাবে দ্বিমত পোষণ করুন এবং কেন তা বুঝিয়ে বলুন।",
        },
        {
          "id": "ST4",
          "title": "দলে যোগদান",
          "desc":
              "একটি চলমান দলগত কথোপকথনে যোগ দিন এবং একটি চিন্তাশীল বাক্য যোগ করুন।",
        },
        {
          "id": "ST5",
          "title": "আলোচনার সূত্রপাত",
          "desc":
              "একটি সামাজিক গ্রুপে আলোচনার জন্য একটি নতুন বিষয় উত্থাপন করুন।",
        },
        {
          "id": "ST6",
          "title": "প্রকাশ্য প্রশ্ন",
          "desc":
              "একটি প্রকাশ্য সভা বা ক্লাসরুমের পরিবেশে কোনো প্রশ্ন জিজ্ঞাসা করুন।",
        },
        {
          "id": "ST7",
          "title": "সাহসী অনুরোধ",
          "desc":
              "কোনো ক্যাফে বা পার্কে কোনো অপরিচিত ব্যক্তিকে জিজ্ঞেস করুন যে আপনি তাঁর পাশে বসতে পারেন কিনা।",
        },
        {
          "id": "ST8",
          "title": "কথোপকথনের সেতু",
          "desc":
              "একে অপরকে চেনে না এমন দুজন মানুষের পরিচয় করিয়ে দিন এবং তাদের মধ্যে একটি সাধারণ মিল খুঁজে বের করুন।",
        },
        {
          "id": "ST9",
          "title": "দৃঢ় প্রয়োজনীয়তা",
          "desc":
              "আপনাকে বিরক্ত করছে এমন কিছু করা বন্ধ করতে বা কাউকে ভদ্রভাবে সরে যেতে বলুন।",
        },
        {
          "id": "ST10",
          "title": "গল্পকথক",
          "desc":
              "৩ বা তার বেশি মানুষের একটি দলের সামনে একটি গল্প শোনানোর ক্ষেত্রে নেতৃত্ব দিন।",
        },
        {
          "id": "ST11",
          "title": "উন্মুক্ত চ্যালেঞ্জ",
          "desc":
              "একটি গ্রুপে একটি সাধারণ মতামতকে একটি বন্ধুত্বপূর্ণ এবং সম্মানজনক উপায়ে চ্যালেঞ্জ করুন।",
        },
        {
          "id": "ST12",
          "title": "সামাজিক উদ্যোগ",
          "desc":
              "একটি ঘরে প্রবেশ করার সময় সবাইকে সবার আগে 'হ্যালো' বলার মতো ব্যক্তি হন।",
        },
        {
          "id": "ST13",
          "title": "সহানুভূতিশীল শ্রবণ",
          "desc":
              "কারও মনের কষ্ট বা ক্ষোভের কথা মনোযোগ দিয়ে শুনুন এবং একটি সহায়ক প্রতিক্রিয়া দিন।",
        },
        {
          "id": "ST14",
          "title": "প্রকাশ্য উপস্থাপনা",
          "desc":
              "একটি সামাজিক সমাবেশে আপনার প্রিয় একটি বিষয় নিয়ে ১-২ মিনিট কথা বলুন।",
        },
        {
          "id": "ST15",
          "title": "দুর্বলতা স্বীকার",
          "desc":
              "একটি দলের সামনে অকপটে স্বীকার করুন যে আপনি কোনো বিষয়ে নার্ভাস ছিলেন, এবং এটি নিয়ে একসাথে হাসুন।",
        },
        {
          "id": "ST16",
          "title": "সীমানা নির্ধারণ",
          "desc":
              "কোনো দীর্ঘ ব্যাখ্যা না দিয়ে, যে আমন্ত্রণে আপনি যেতে চান না তা ভদ্রভাবে প্রত্যাখ্যান করুন।",
        },
        {
          "id": "ST17",
          "title": "সক্রিয় মধ্যস্থতাকারী",
          "desc":
              "একটি মতবিরোধে দুজন মানুষকে একটি মাঝামাঝি মীমাংসায় আসতে সাহায্য করুন।",
        },
        {
          "id": "ST18",
          "title": "প্রকাশ্য প্রশংসা",
          "desc":
              "একটি গ্রুপে প্রকাশ্যভাবে কারও প্রচেষ্টা বা অর্জনের প্রশংসা করুন।",
        },
        {
          "id": "ST19",
          "title": "সরাসরি পদ্ধতি",
          "desc":
              "আপনার প্রয়োজন এমন কোনো সাহায্য বা উপদেশের জন্য কাউকে সরাসরি অনুরোধ করুন।",
        },
        {
          "id": "ST20",
          "title": "কথোপকথনের মোড় ঘোরানো",
          "desc":
              "একটি কথোপকথনকে একটি বিরক্তিকর বিষয় থেকে একটি আকর্ষণীয় বিষয়ে মসৃণভাবে স্থানান্তরিত করুন।",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "উপহার",
          "desc":
              "কাউকে একটি ছোট উপহার বা খাওয়ার জিনিস দিন এবং বলুন 'আমি ভেবেছিলাম এটি আপনার পছন্দ হবে'।",
        },
        {
          "id": "B2",
          "title": "সাহসী নেতৃত্ব",
          "desc":
              "মানুষের একটি ছোট গ্রুপকে একটি পরিকল্পনা বা কোনো জায়গায় ঘুরতে যাওয়ার প্রস্তাব দিন।",
        },
        {
          "id": "B3",
          "title": "কৃতজ্ঞতা প্রকাশ",
          "desc":
              "কাউকে নির্দিষ্টভাবে বলুন যে কেন আপনি আপনার জীবনে তাঁদের উপস্থিতি মূল্যবান মনে করেন।",
        },
        {
          "id": "B4",
          "title": "সামাজিক সংগঠক",
          "desc":
              "কয়েকজন মানুষের জন্য একটি ছোট মিলনমেলা বা কফি ডেটের আয়োজন করুন।",
        },
        {
          "id": "B5",
          "title": "গভীর কথোপকথন",
          "desc":
              "কারও সাথে ১৫ মিনিটেরও বেশি সময় ধরে একটি গভীর এবং অর্থপূর্ণ কথোপকথন করুন।",
        },
        {
          "id": "B6",
          "title": "আত্মবিশ্বাসের চূড়া",
          "desc":
              "এমন কারও সাথে কথোপকথন শুরু করুন যাকে দেখে আপনি কিছুটা সংকোচ বা ভয় বোধ করেন।",
        },
        {
          "id": "B7",
          "title": "প্রকাশ্য অভিনন্দন",
          "desc":
              "একটি গ্রুপে কোনো ব্যক্তির জন্য একটি সংক্ষিপ্ত, ইতিবাচক প্রশংসাসূচক বাক্য বলুন বা তাদের প্রচেষ্টাকে স্বাগত জানান।",
        },
        {
          "id": "B8",
          "title": "সীমা নির্ধারণকারী",
          "desc":
              "কোনো দীর্ঘ ব্যাখ্যা ছাড়াই, কোনো অনুরোধে দৃঢ়ভাবে কিন্তু বিনীতভাবে 'না' বলুন।",
        },
        {
          "id": "B9",
          "title": "সরাসরি অনুরোধ",
          "desc":
              "আপনি পছন্দ করেন বা শ্রদ্ধা করেন এমন একজনের কাছে ১০ মিনিটের আলাপ বা দিকনির্দেশনার অনুরোধ করুন।",
        },
        {
          "id": "B10",
          "title": "আবেগীয় উদ্যোগ",
          "desc":
              "আপনার কোনো বন্ধুর সাথে অনুভূতি বা মানসিক স্বাস্থ্যের বিষয়ে গভীর আলোচনার সূত্রপাত করুন।",
        },
        {
          "id": "B11",
          "title": "সামাজিক মধ্যস্থতাকারী",
          "desc":
              "শান্ত কথোপকথনের মাধ্যমে দুজন মানুষের মধ্যে একটি ছোট বিরোধের সমাধান করতে সহায়তা করুন।",
        },
        {
          "id": "B12",
          "title": "সাহসী প্রশংসা",
          "desc":
              "সম্পূর্ণ অপরিচিত কাউকে এমন একটি কথা বলুন যা আপনি সত্যিই তাঁদের মধ্যে পছন্দ বা প্রশংসা করেন।",
        },
        {
          "id": "B13",
          "title": "নেটওয়ার্কিং পদক্ষেপ",
          "desc":
              "আপনার ক্ষেত্রের একজন পেশাদার বা বিশেষজ্ঞ ব্যক্তির সাথে নিজের পরিচয় করিয়ে দিন এবং পরামর্শ চান।",
        },
        {
          "id": "B14",
          "title": "সাহসী সত্য",
          "desc":
              "কাউকে এমন একটি সত্য কথা বলুন যা বলা কঠিন কিন্তু সম্পর্ক বা বন্ধনের জন্য উপকারী।",
        },
        {
          "id": "B15",
          "title": "পূর্ণ প্রস্ফুটন",
          "desc":
              "একটি ছোট সামাজিক অনুষ্ঠানের আয়োজন করুন এবং নিশ্চিত করুন যে প্রতিটি অতিথি যেন স্বাচ্ছন্দ্য বোধ করেন।",
        },
        {
          "id": "B16",
          "title": "প্রকাশ্য বক্তা",
          "desc":
              "কোনো মিটিং বা অনুষ্ঠানের একটি ছোট অংশ পরিচালনা করার বা কথা বলার জন্য নিজে স্বেচ্ছায় এগিয়ে আসুন।",
        },
        {
          "id": "B17",
          "title": "অভিজ্ঞতার আলোয় গাইড",
          "desc":
              "অন্য কাউকে উৎসাহিত করার জন্য আপনার কোনো কঠিন সময় বা কাটিয়ে ওঠা ব্যর্থতার অভিজ্ঞতা শেয়ার করুন।",
        },
        {
          "id": "B18",
          "title": "সাহসী ক্ষমা প্রার্থনা",
          "desc":
              "অতীতের কোনো ভুলের জন্য ক্ষমা চাইতে নিজেই কথোপকথনের উদ্যোগ নিন, তা যতই পুরনো হোক না কেন।",
        },
        {
          "id": "B19",
          "title": "পরামর্শদাতা (মেন্টর)",
          "desc":
              "এমন কোনো ব্যক্তিকে কোনো দক্ষতা বা কাজে সাহায্যের প্রস্তাব দিন যিনি আপনার চেয়ে কম অভিজ্ঞ।",
        },
        {
          "id": "B20",
          "title": "সামাজিক স্থপতি",
          "desc":
              "বন্ধুদের একটি গ্রুপের জন্য একটি নতুন সামাজিক ঐতিহ্য বা বারবার হওয়া কোনো আড্ডা (মিটআপ)-এর সূচনা করুন।",
        },
      ],
    },
    'ta': {
      "Seedling": [
        {
          "id": "S1",
          "title": "முதல் அடி",
          "desc": "இன்று ஒரு நபருடன் கண் தொடர்பு கொண்டு புன்னகைக்கவும்.",
        },
        {
          "id": "S2",
          "title": "ஒரு எளிய வணக்கம்",
          "desc":
              "பக்கத்து வீட்டுக்காரரிடம் 'காலை வணக்கம்' அல்லது 'வணக்கம்' என்று கூறவும்.",
        },
        {
          "id": "S3",
          "title": "நன்றி கூறுதல்",
          "desc": "ஒரு கடைக்காரரிடம் தெளிவாக 'நன்றி' என்று கூறவும்.",
        },
        {
          "id": "S4",
          "title": "கூர்ந்து நோக்குதல்",
          "desc":
              "அந்நியர் ஒருவரிடம் உள்ள ஒரு நேர்மறையான விஷயத்தைக் கவனித்து புன்னகைக்கவும்.",
        },
        {
          "id": "S5",
          "title": "அமைதியான கை அசைவு",
          "desc":
              "தொலைவில் உங்களுக்குத் தெரிந்த ஒருவரைப் பார்த்தால் கை அசைக்கவும்.",
        },
        {
          "id": "S6",
          "title": "கதவை பிடித்துக் கொள்ளுதல்",
          "desc":
              "உங்களுக்குப் பின்னால் வருபவருக்காகக் கதவைத் திறந்து பிடித்துக் கொள்ளவும்.",
        },
        {
          "id": "S7",
          "title": "தலை அசைப்பு",
          "desc":
              "கடந்து செல்லும்போது சக ஊழியருக்குப் புன்னகையுடன் தலை அசைத்து வணக்கம் கூறவும்.",
        },
        {
          "id": "S8",
          "title": "கண்ணாடிப் பயிற்சி",
          "desc":
              "கண்ணாடி முன் நின்று 1 நிமிடம் உங்கள் 'தன்னம்பிக்கை புன்னகையை' பயிற்சி செய்யவும்.",
        },
        {
          "id": "S9",
          "title": "சுருக்கமான பார்வை",
          "desc":
              "ஒருவரை 2 வினாடிகள் பார்த்து, பின் புன்னகைத்து பார்வையைத் திருப்பிக் கொள்ளவும்.",
        },
        {
          "id": "S10",
          "title": "அமைதியான பாராட்டு",
          "desc":
              "சமூக ஊடகத்தில் ஒருவரின் பதிவின் கீழ் ஒரு நல்ல கருத்தை (Comment) எழுதவும்.",
        },
        {
          "id": "S11",
          "title": "இடத்தைப் பகிர்ந்து கொள்ளுதல்",
          "desc":
              "பொது இடத்தில் ஒருவருக்கு அருகில் அமர்ந்து, உடனே பார்வையைத் திருப்பாமல் சில நிமிடம் இருக்கவும்.",
        },
        {
          "id": "S12",
          "title": "எளிய அனுமதி",
          "desc":
              "பின்னடைவாரத்தில் ஒருவரைக் கடந்து செல்லும்போது கனிவுடன் 'மன்னிக்கவும்' என்று கூறவும்.",
        },
        {
          "id": "S13",
          "title": "அன்பான வரவேற்பு",
          "desc":
              "டெலிவரி செய்பவரிடம் அல்லது தபால்காரரிடம் 'வணக்கம்' என்று கூறவும்.",
        },
        {
          "id": "S14",
          "title": "சிறிய சைகை",
          "desc":
              "ஒரு குழந்தைக்கு அல்லது (உரிமையாளரின் அனுமதியுடன்) ஒரு செல்லப் பிராணிக்குக் கை அசைக்கவும்.",
        },
        {
          "id": "S15",
          "title": "மென்மையான புன்னகை",
          "desc": "இன்று மூன்று வெவ்வேறு நபர்களைப் பார்த்துப் புன்னகைக்கவும்.",
        },
        {
          "id": "S16",
          "title": "கண் தொடர்பு சவால்",
          "desc":
              "பணப் பதிவேட்டாளர் (Cashier) முதலில் பார்வையைத் திருப்பும் வரை அவருடன் கண் தொடர்பைப் பேணவும்.",
        },
        {
          "id": "S17",
          "title": "அமைதியான சுவாசம்",
          "desc":
              "இன்று ஏதேனும் சமூக இடத்திற்குள் நுழைவதற்கு முன் 3 முறை ஆழமாக மூச்சை இழுத்து விடவும்.",
        },
        {
          "id": "S18",
          "title": "இருப்பு",
          "desc":
              "கூட்ட நெரிசலான ஒரு இடத்தில் உங்கள் தொலைபேசியைப் பார்க்காமல் 5 நிமிடங்கள் நிற்கவும்.",
        },
        {
          "id": "S19",
          "title": "இயல்பான தலை அசைப்பு",
          "desc":
              "உங்களுடன் கண் தொடர்பு கொள்ளும் ஒரு அந்நியருக்குத் தலை அசைத்து வணக்கம் தெரிவிக்கவும்.",
        },
        {
          "id": "S20",
          "title": "இனிய விடைபெறல்",
          "desc":
              "ஒரு கடையை விட்டு வெளியேறும்போது ஒருவரிடம் 'இந்த நாள் இனிய நாளாக அமையட்டும்' என்று கூறவும்.",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "பாராட்டுதல்",
          "desc":
              "உடன் பணிபுரிபவர் அல்லது சக மாணவர் ஒருவரை மனதாரப் பாராட்டவும்.",
        },
        {
          "id": "SP2",
          "title": "கேள்வி கேட்டல்",
          "desc": "அந்நியர் ஒருவரிடம் நேரம் அல்லது வழியைக் கேட்கவும்.",
        },
        {
          "id": "SP3",
          "title": "சிறு உரையாடல்",
          "desc":
              "ஒருவரிடம் 'உங்கள் நாள் எப்படிச் செல்கிறது?' என்று கேட்டு, அவர்களின் பதிலைக் கூர்ந்து கவனிக்கவும்.",
        },
        {
          "id": "SP4",
          "title": "உதவி கோருதல்",
          "desc":
              "கடை ஊழியர் ஒருவரிடம் ஒரு குறிப்பிட்ட பொருளைக் கண்டறிய உதவி கேட்கவும்.",
        },
        {
          "id": "SP5",
          "title": "ஆர்டர் செய்யும்போது",
          "desc":
              "உணவு அல்லது பானம் ஆர்டர் செய்யும்போது அங்கிருக்கும் ஊழியர்களிடம் நலம் விசாரிக்கவும்.",
        },
        {
          "id": "SP6",
          "title": "அறிமுகம் செய்தல்",
          "desc":
              "உங்கள் பகுதியில் உள்ள ஒரு புதிய நபரிடம் உங்களை அறிமுகப்படுத்திக் கொள்ளவும்.",
        },
        {
          "id": "SP7",
          "title": "வானிலை உரையாடல்",
          "desc":
              "வரிசையில் காத்திருக்கும் போது அருகில் உள்ளவரிடம் வானிலை பற்றிப் பேசவும்.",
        },
        {
          "id": "SP8",
          "title": "எளிய நலம் விசாரிப்பு",
          "desc":
              "சக ஊழியரிடம் 'வார இறுதியில் என்ன செய்தீர்கள்?' என்று கேட்கவும்.",
        },
        {
          "id": "SP9",
          "title": "உதவி செய்தல்",
          "desc":
              "யாராவது சிரமப்படுவதைக் கண்டால், 'நான் உங்களுக்கு உதவட்டுமா?' என்று கேட்கவும்.",
        },
        {
          "id": "SP10",
          "title": "கருத்து அறிதல்",
          "desc":
              "ஒரு நண்பரிடம் ஒரு சிறிய பொருளைக் காட்டி, 'இதைப் பற்றி நீங்கள் என்ன நினைக்கிறீர்கள்?' என்று கேட்கவும்.",
        },
        {
          "id": "SP11",
          "title": "உறுதிப்படுத்துதல்",
          "desc":
              "அந்நியரிடம் ஒரு விவரத்தை உறுதிப்படுத்தவும் (எ.கா: 'இதுதான் சரியான வரிசையா?').",
        },
        {
          "id": "SP12",
          "title": "சூழல் பற்றி",
          "desc":
              "சுற்றியுள்ள சூழலைப் பற்றி ஒரு சிறிய கருத்து கூறவும் (எ.கா: 'இங்கு கூட்டம் மிகவும் அதிகமாக உள்ளது').",
        },
        {
          "id": "SP13",
          "title": "சிறிய உதவி",
          "desc":
              "மேஜையில் இருக்கும் போது ஒரு பொருளை (நாப்கின் போன்றவற்றை) உங்களிடம் நகர்த்திக் கொடுக்குமாறு கேட்கவும்.",
        },
        {
          "id": "SP14",
          "title": "நல்ல கருத்துரை",
          "desc":
              "வெளியேறுவதற்கு முன் உணவக ஊழியரிடம் உணவு மிகவும் அருமையாக இருந்தது என்று கூறவும்.",
        },
        {
          "id": "SP15",
          "title": "சாதாரணத் தொடர்பு",
          "desc":
              "ஒரு மாதமாகப் பேசாத ஒருவருக்கு 'எப்படி இருக்கிறீர்கள்?' என்று ஒரு குறுஞ்செய்தி அனுப்பவும்.",
        },
        {
          "id": "SP16",
          "title": "திறந்த கேள்வி",
          "desc":
              "ஒருவரிடம் 'இந்த நகரத்தில் உங்களுக்கு மிகவும் பிடித்தமான சுற்றும் இடம் எது?' என்று கேட்கவும்.",
        },
        {
          "id": "SP17",
          "title": "மிகச் சிறிய முயற்சி",
          "desc":
              "அந்நியர் ஒருவரிடம் மிக அருகில் உள்ள கழிவறை எங்குள்ளது என்று கேட்கவும்.",
        },
        {
          "id": "SP18",
          "title": "பொருட்களின் பாராட்டு",
          "desc":
              "ஒருவரிடம் அவர்களின் காலணிகள்/பை/அணிகலன் நன்றாக உள்ளது என்று கூறவும்.",
        },
        {
          "id": "SP19",
          "title": "மரியாதையான இடைவெளி",
          "desc":
              "ஒருவருக்குப் பதிலளிப்பதற்கு முன் அவரின் பேச்சு முழுமையாக முடியும் வரை பொறுமையாகக் காத்திருக்கவும்.",
        },
        {
          "id": "SP20",
          "title": "அன்பான விடைபெறல்",
          "desc":
              "சற்று முன் உங்களுடன் சுருக்கமாகப் பேசிய ஒருவருக்குக் கை அசைத்து 'டாடா' கூறவும்.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "கருத்துத் தேடுபவர்",
          "desc":
              "ஒரு புத்தகம், திரைப்படம் அல்லது பாடல் பற்றிய அவர்களின் கருத்தைக் கேட்கவும்.",
        },
        {
          "id": "L2",
          "title": "கூடுதல் விவரம்",
          "desc":
              "ஒருவர் தங்களைப் பற்றி ஏதேனும் கூறிய பிறகு, அது தொடர்பான மற்றொரு கேள்வியைக் கேட்கவும்.",
        },
        {
          "id": "L3",
          "title": "பரிந்துரை",
          "desc":
              "அந்நியரிடம் அருகில் உள்ள ஒரு நல்ல உணவகத்தைப் பற்றிப் பரிந்துரை கேட்கவும்.",
        },
        {
          "id": "L4",
          "title": "பொதுவான இணைப்பு",
          "desc":
              "ஒருவருடன் ஒரு பொதுவான ஆர்வத்தைக் கண்டறிந்து, அதைப் பற்றி 2 நிமிடங்கள் பேசவும்.",
        },
        {
          "id": "L5",
          "title": "உதவிக் கரம்",
          "desc":
              "ஒரு சிறிய வேலையில் (பை தூக்குவது போன்ற) உதவி செய்ய முன்வரவும்.",
        },
        {
          "id": "L6",
          "title": "சமூகக் கவனிப்பு",
          "desc":
              "உங்கள் இருவரையும் சுற்றி நடக்கும் ஒரு நிகழ்வை அடிப்படையாகக் கொண்டு உரையாடலைத் தொடங்கவும்.",
        },
        {
          "id": "L7",
          "title": "திறந்த கேள்வி",
          "desc":
              "ஒருவரிடம் 'நீங்கள் இந்த வேலைத் துறைக்குள் எப்படி வந்தீர்கள்?' என்று கேட்கவும்.",
        },
        {
          "id": "L8",
          "title": "தீவிரமாகக் கேட்ப்பவர்",
          "desc":
              "ஒருவரின் பேச்சைத் குறுக்கிடாமல் 3 நிமிடங்கள் கேட்கவும், பின் அவர் கூறியதைச் சுருக்கிக் கூறவும்.",
        },
        {
          "id": "L9",
          "title": "பொதுவான நகைச்சுவை",
          "desc":
              "ஒரு சிறிய குழுவிற்கு ஒரு சிறிய, நகைச்சுவையான கதை அல்லது ஜோக் கூறவும்.",
        },
        {
          "id": "L10",
          "title": "ஆர்வம்",
          "desc":
              "ஒருவரிடம் அவர் எந்த ஊர் என்றும், அந்த ஊரில் அவருக்கு என்ன பிடிக்கும் என்றும் கேட்கவும்.",
        },
        {
          "id": "L11",
          "title": "உண்மையான அக்கறை",
          "desc":
              "சக ஊழியரிடம் வேலைக்கு அப்பாற்பட்ட அவரின் பொழுதுபோக்குகளைப் பற்றிக் கேட்கவும்.",
        },
        {
          "id": "L12",
          "title": "மென்மையான ஆலோசனை",
          "desc":
              "நீங்கள் நிபுணராக இருக்கும் ஒரு விஷயத்தைப் பற்றி ஒருவருக்கு பயனுள்ள ஆலோசனையைக் கூறவும்.",
        },
        {
          "id": "L13",
          "title": "குழுவில் ஆதரவு",
          "desc":
              "ஒரு சிறிய குழு விவாதத்தில் ஒருவரின் கருத்தை ஆதரித்துத் தலை அசைக்கவும்.",
        },
        {
          "id": "L14",
          "title": "சாதாரண அழைப்பு",
          "desc":
              "ஒருவரிடம் 'மதிய உணவிற்கு எங்களுடன் சேர்ந்து கொள்ள விரும்புகிறீர்களா?' என்று கேட்கவும்.",
        },
        {
          "id": "L15",
          "title": "உண்மையான வெளிப்பாடு",
          "desc":
              "ஒருவரிடம் 'நீங்கள் எக்ஸ் (X) செய்தபோது எனக்கு மிகவும் மகிழ்ச்சியாக இருந்தது' என்று கூறி அதற்கான காரணத்தை விளக்கவும்.",
        },
        {
          "id": "L16",
          "title": "ஆர்வ இடைவெளி",
          "desc":
              "ஒருவரிடம் 'நான் எப்போதும் யோசிப்பேன், எக்ஸ் (X) உண்மையில் எப்படி வேலை செய்கிறது?' என்று கேட்கவும்.",
        },
        {
          "id": "L17",
          "title": "சிறு குழு வழிகாட்டி",
          "desc":
              "ஒரு குழுவில் 2 அல்லது 3 பேர் பதிலளிக்க வேண்டிய ஒரு கேள்வியைக் கேட்கவும்.",
        },
        {
          "id": "L18",
          "title": "உண்மையான பாராட்டு",
          "desc":
              "ஒருவரின் குணநலனைப் பாராட்டவும் (எ.கா: 'நீங்கள் ஒரு சிறந்த கேட்பவர்').",
        },
        {
          "id": "L19",
          "title": "பகிரப்பட்ட அனுபவம்",
          "desc":
              "உரையாடலின் போது 'நானும் அந்தச் சூழ்நிலையில் இருந்திருக்கிறேன்' என்று கூறவும்.",
        },
        {
          "id": "L20",
          "title": "பொருளுள்ள இடைவெளி",
          "desc":
              "உரையாடலில் அமைதியான தருணம் ஏற்பட்டால் அதை அவசரமாக நிரப்ப முயலாமல் இயல்பாக இருக்கவும்.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "துணிச்சலான ஆரம்பம்",
          "desc":
              "உங்களுக்கு நன்றாகத் தெரியாத ஒருவருடன் உரையாடலைத் தொடங்கவும்.",
        },
        {
          "id": "ST2",
          "title": "நேர்மையான பகிர்வு",
          "desc":
              "ஒரு குழு அமைப்பில் ஒரு சிறிய தனிப்பட்ட கதை அல்லது கருத்தைப் பகிரவும்.",
        },
        {
          "id": "ST3",
          "title": "விவாதம்",
          "desc":
              "ஒருவரின் கருத்துடன் மரியாதையுடன் மாறுபட்டு, அதற்கான காரணத்தை விளக்கவும்.",
        },
        {
          "id": "ST4",
          "title": "குழுவில் இணைதல்",
          "desc":
              "ஒரு குழு உரையாடலில் இணைந்து, ஒரு பயனுள்ள வாக்கியத்தைச் சேர்க்கவும்.",
        },
        {
          "id": "ST5",
          "title": "தலைப்பைத் தொடங்குதல்",
          "desc":
              "ஒரு சமூகக் குழுவில் விவாதத்திற்கு ஒரு புதிய தலைப்பை அறிமுகப்படுத்தவும்.",
        },
        {
          "id": "ST6",
          "title": "பொதுவான கேள்வி",
          "desc":
              "ஒரு பொதுக் கூட்டம் அல்லது வகுப்பறைச் சூழலில் ஒரு கேள்வி கேட்கவும்.",
        },
        {
          "id": "ST7",
          "title": "துணிச்சலான கோரிக்கை",
          "desc":
              "ஒரு கஃபே அல்லது பூங்காவில் ஒரு அந்நியரிடம் அவரின் அருகில் அமரலாமா என்று கேட்கவும்.",
        },
        {
          "id": "ST8",
          "title": "உரையாடல் பாலம்",
          "desc":
              "ஒருவரையொருவர் அறியாத இருவரை அறிமுகப்படுத்தி, அவர்களுக்கு இடையே ஒரு பொதுவான தன்மையைக் கண்டறியவும்.",
        },
        {
          "id": "ST9",
          "title": "உறுதியான தேவை",
          "desc":
              "உங்களை தொந்தரவு செய்யும் ஒரு விஷயத்தை நிறுத்துமாறு அல்லது ஒருவரை நகர்ந்து செல்லுமாறு மரியாதையுடன் கூறவும்.",
        },
        {
          "id": "ST10",
          "title": "கதைசொல்லி",
          "desc":
              "3 அல்லது அதற்கு மேற்பட்ட நபர்களைக் கொண்ட குழுவிற்கு ஒரு கதையைக் கூறுவதில் தலைமை தாங்கவும்.",
        },
        {
          "id": "ST11",
          "title": "திறந்த சவால்",
          "desc":
              "ஒரு குழுவில் உள்ள பொதுவான கருத்தை ஒரு நட்பு மற்றும் மரியாதையான வழியில் சவால் செய்யவும்.",
        },
        {
          "id": "ST12",
          "title": "சமூக முயற்சி",
          "desc":
              "ஒரு அறைக்குள் நுழையும் போது அனைவருக்கும் முதலில் 'வணக்கம்' கூறும் நபராக இருக்கவும்.",
        },
        {
          "id": "ST13",
          "title": "உணர்வுகளைக் கேட்டல்",
          "desc":
              "ஒருவர் தனது மனக்குறையைக் கூறும்போது அதைக் கேட்டு, ஆதரவான பதிலை அளிக்கவும்.",
        },
        {
          "id": "ST14",
          "title": "பொது வெளியீடு",
          "desc":
              "ஒரு சமூகக் கூட்டத்தில் உங்களுக்குப் பிடித்த ஒரு தலைப்பைப் பற்றி 1-2 நிமிடங்கள் பேசவும்.",
        },
        {
          "id": "ST15",
          "title": "பலவீனத்தை ஒப்புக்கொள்ளுதல்",
          "desc":
              "ஒரு குழுவின் முன் நீங்கள் ஏதோ ஒரு விஷயத்தில் பதற்றமாக இருந்ததை ஒப்புக்கொண்டு, அதைப் பற்றி ஒன்றாக சிரிக்கவும்.",
        },
        {
          "id": "ST16",
          "title": "எல்லைகளை வகுத்தல்",
          "desc":
              "எந்தவொரு நீண்ட விளக்கமும் தராமல், நீங்கள் செல்ல விரும்பாத அழைப்பை மரியாதையுடன் நிராகரிக்கவும்.",
        },
        {
          "id": "ST17",
          "title": "சক্রিয় மத்தியஸ்தர்",
          "desc":
              "ஒரு கருத்து வேறுபாட்டில் இரு நபர்களை ஒரு நடுநிலையான முடிவுக்கு வர உதவவும்.",
        },
        {
          "id": "ST18",
          "title": "பொதுவான பாராட்டு",
          "desc":
              "ஒரு குழுவில் ஒருவரின் முயற்சி அல்லது சாதனையைப் பகிரங்கமாகப் பாராட்டவும்.",
        },
        {
          "id": "ST19",
          "title": "நேரடி அணுகுமுறை",
          "desc":
              "உங்களுக்குத் தேவையான ஒரு உதவி அல்லது ஆலோசனைக்காக ஒருவரிடம் நேரடியாகக் கோரிக்கை வைக்கவும்.",
        },
        {
          "id": "ST20",
          "title": "உரையாடலின் திருப்பம்",
          "desc":
              "ஒரு உரையாடலை ஒரு சலிப்பான தலைப்பிலிருந்து சுவாரஸ்யமான தலைப்பிற்கு மென்மையாக மாற்றவும்.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "பரிசு",
          "desc":
              "காருக்கு ஒரு சிறிய பரிசு அல்லது திண்பண்டம் கொடுத்து 'இது உங்களுக்குப் பிடிக்கும் என்று நினைத்தேன்' என்று கூறவும்.",
        },
        {
          "id": "B2",
          "title": "துணிச்சலான தலைமை",
          "desc":
              "மனிதர்களின் ஒரு சிறிய குழுவிற்கு ஒரு திட்டம் அல்லது ஏதேனும் இடத்திற்குச் செல்லும் யோசனையை முன்வைக்கவும்.",
        },
        {
          "id": "B3",
          "title": "மதிப்பு வெளிப்பாடு",
          "desc":
              "ஒருவரிடம் குறிப்பிட்டுக் கூறவும், ஏன் உங்கள் வாழ்வில் அவர்களின் இருப்பை நீங்கள் மதிப்புமிக்கதாகக் கருதுகிறீர்கள் என்று.",
        },
        {
          "id": "B4",
          "title": "சமூக அமைப்பாளர்",
          "desc":
              "சில நபர்களுக்காக ஒரு சிறிய சந்திப்பு அல்லது ஒரு காபி டேட்டிற்கு ஏற்பாடு செய்யவும்.",
        },
        {
          "id": "B5",
          "title": "ஆழ்ந்த உரையாடல்",
          "desc":
              "ஒருவருடன் 15 நிமிடங்களுக்கும் மேலாக ஆழமான மற்றும் அர்த்தமுள்ள உரையாடலை மேற்கொள்ளவும்.",
        },
        {
          "id": "B6",
          "title": "தன்னம்பிக்கையின் சிகரம்",
          "desc":
              "உங்களைப் பார்க்கும்போது சற்று பயம் அல்லது தயக்கம் ஏற்படும் ஒருவருடன் உரையாடலைத் தொடங்கவும்.",
        },
        {
          "id": "B7",
          "title": "பொதுவான வாழ்த்து",
          "desc":
              "ஒரு குழுவில் ஒரு குறிப்பிட்ட நபருக்காக ஒரு சிறிய, நேர்மறையான வாழ்த்து வாக்கியத்தைக் கூறவும் அல்லது அவரின் முயற்சியை வரவேற்கவும்.",
        },
        {
          "id": "B8",
          "title": "எல்லை வகுப்பவர்",
          "desc":
              "எந்தவொரு நீண்ட விளக்கமும் இன்றி, ஒரு கோரிக்கைக்கு உறுதியாக ஆனால் கனிவுடன் 'இல்லை' என்று கூறவும்.",
        },
        {
          "id": "B9",
          "title": "நேரடி கோரிக்கை",
          "desc":
              "நீங்கள் விரும்பும் அல்லது மதிக்கும் ஒருவரிடம் 10 நிமிட பேச்சு அல்லது வழிகாட்டுதலுக்கான கோரிக்கை வைக்கவும்.",
        },
        {
          "id": "B10",
          "title": "உணர்ச்சிப்பூர்வமான முயற்சி",
          "desc":
              "உங்கள் நண்பர் ஒருவருடன் உணர்வுகள் அல்லது மன ஆரோக்கியம் குறித்த ஆழமான விவாதத்தைத் தொடங்கவும்.",
        },
        {
          "id": "B11",
          "title": "சமூக மத்தியஸ்தர்",
          "desc":
              "அமைதியான உரையாடலின் மூலம் இரு நபர்களுக்கு இடையிலான ஒரு சிறிய மோதலைத் தீர்க்க உதவவும்.",
        },
        {
          "id": "B12",
          "title": "துணிச்சலான பாராட்டு",
          "desc":
              "முற்றிலும் அந்நியரான ஒருவரிடம் நீங்கள் உண்மையாகவே அவரிடம் பாராட்டும் ஒரு விஷயத்தைக் கூறவும்.",
        },
        {
          "id": "B13",
          "title": "நெറ്റ്‌വർக்கிங் நடவடிக்கை",
          "desc":
              "உங்கள் துறையைச் சேர்ந்த ஒரு நிபுணர் அல்லது தகுதிவாய்ந்த நபரிடம் உங்களை அறிமுகப்படுத்திக் கொண்டு ஆலோசனை கேட்கவும்.",
        },
        {
          "id": "B14",
          "title": "துணிச்சலான உண்மை",
          "desc":
              "ஒருவரிடம் கூறுவதற்கு கடினமான ஆனால் உறவு அல்லது பிணைப்பிற்குப் பயனுள்ள ஒரு உண்மையைச் கூறவும்.",
        },
        {
          "id": "B15",
          "title": "முழு மலர்ச்சி",
          "desc":
              "ஒரு சிறிய சமூக நிகழ்ச்சியை ஏற்பாடு செய்து, அங்கு வரும் ஒவ்வொரு விருந்தினரும் வசதியாக இருப்பதை உறுதி செய்யவும்.",
        },
        {
          "id": "B16",
          "title": "பொதுப் பேச்சாளர்",
          "desc":
              "ஒரு கூட்டம் அல்லது நிகழ்ச்சியின் ஒரு சிறிய பகுதியை வழிநடத்த அல்லது பேசுவதற்கு நீங்களாகவே முன்வரவும்.",
        },
        {
          "id": "B17",
          "title": "அனுபவ வழிகாட்டி",
          "desc":
              "மற்றொருவரை ஊக்குவிப்பதற்காக உங்கள் கடினமான சூழ்நிலை அல்லது நீங்கள் கடந்த தோல்வியின் அனுபவத்தைப் பகிரவும்.",
        },
        {
          "id": "B18",
          "title": "துணிச்சலான மன்னிப்பு",
          "desc":
              "கடந்த கால ஒரு தவறுக்காக மன்னிப்பு கேட்க நீங்களாகவே உரையாடலைத் தொடங்கவும், அது எவ்வளவு பழமையானதாக இருந்தாலும்.",
        },
        {
          "id": "B19",
          "title": "ஆலோசகர் (Mentor)",
          "desc":
              "உங்களை விட அனுபவத்தில் குறைந்த ஒரு நபருக்கு ஒரு திறன் அல்லது வேலையில் உதவி செய்ய முன்வரவும்.",
        },
        {
          "id": "B20",
          "title": "சமூகக் கலைஞர்",
          "desc":
              "நண்பர்கள் குழுவிற்காக ஒரு புதிய சமூகப் பாரம்பரியம் அல்லது அடிக்கடி நிகழும் ஒரு சந்திப்பைத் (Meetup) தொடங்கவும்.",
        },
      ],
    },
    'te': {
      "Seedling": [
        {
          "id": "S1",
          "title": "మొదటి అడుగు",
          "desc": "ఈ రోజు ఒకరితో కళ్ళు కలిపి, చూసి చిరునవ్వు నవ్వు.",
        },
        {
          "id": "S2",
          "title": "ఒక సాధారణ హలో",
          "desc": "పక్కింటి వారితో 'శుభోదయం' లేదా 'నమస్తే' అని చెప్పు.",
        },
        {
          "id": "S3",
          "title": "ధన్యవాదాలు",
          "desc": "దుకాణదారుడికి స్పష్టంగా 'ధన్యవాదాలు' అని చెప్పు.",
        },
        {
          "id": "S4",
          "title": "పరిశీలన",
          "desc":
              "ఒక అపరిచిత వ్యక్తిలో ఏదైనా మంచి విషయాని గమనించి చిరునవ్వు నవ్వు.",
        },
        {
          "id": "S5",
          "title": "నిశ్శబ్ద హస్తచాలనం",
          "desc": "దూరం నుండి నువ్వు గుర్తుపట్టిన ఎవరికైనా చిన్నగా చేయి ఊపు.",
        },
        {
          "id": "S6",
          "title": "తలుపు పట్టుకోవడం",
          "desc": "నీ వెనుక వస్తున్న వారి కోసం తలుపు తీసి పట్టుకో.",
        },
        {
          "id": "S7",
          "title": "తల ఊపడం",
          "desc":
              "పక్కనుండి వెళ్తున్నప్పుడు తోటి ఉద్యోగికి స్నేహపూర్వకంగా తల ఊపి నమస్కరించు.",
        },
        {
          "id": "S8",
          "title": "అద్దం ప్రాక్టీస్",
          "desc":
              "అద్దం ముందు నిలబడి 1 నిమిషం పాటు నీ 'ఆత్మవిశ్వాస చిరునవ్వును' ప్రాక్టీస్ చేయి.",
        },
        {
          "id": "S9",
          "title": "చిన్న చూపు",
          "desc": "ఎవరినైనా 2 సెకన్ల పాటు చూసి, చిరునవ్వు నవ్వి చూపు తిప్పుకో.",
        },
        {
          "id": "S10",
          "title": "నిశ్శబ్ద ప్రశంస",
          "desc": "ఎవరిదైనా సోషల్ మీడియా పోస్ట్ కింద ఒక మంచి కామెంట్ రాయి.",
        },
        {
          "id": "S11",
          "title": "స్థలాన్ని పంచుకోవడం",
          "desc":
              "పబ్లిక్ ఏరియాలో ఎవరికైనా పక్కన కూర్చో మరియు వెంటనే చూపు తిప్పుకోకుండా ఉండు.",
        },
        {
          "id": "S12",
          "title": "సాధారణ మర్యాద",
          "desc":
              "కారిడార్‌లో ఎవరినైనా దాటి వెళ్తున్నప్పుడు మర్యాదగా 'ఎక్స్‌క్యూజ్ మీ' అని చెప్పు.",
        },
        {
          "id": "S13",
          "title": "ఆత్మీయ పలకరింపు",
          "desc": "డెలివరీ బాయ్ లేదా కొరియర్ వ్యక్తికి 'నమస్తే' అని చెప్పు.",
        },
        {
          "id": "S14",
          "title": "చిన్న సైగ",
          "desc":
              "ఒక చిన్న బిడ్డకు లేదా (యజమాని అనుమతితో) పెంపుడు జంతువుకు చేయి ఊపు.",
        },
        {
          "id": "S15",
          "title": "మృదువైన చిరునవ్వు",
          "desc": "ఈ రోజు ముగ్గురు వేర్వేరు వ్యక్తులను చూసి చిరునవ్వు నవ్వు.",
        },
        {
          "id": "S16",
          "title": "ఐ కాంటాక్ట్ సవాలు",
          "desc":
              "క్యాషియర్ మొదట చూపు తిప్పుకునే వరకు వారితో కళ్ళు కలిపి ఉంచు.",
        },
        {
          "id": "S17",
          "title": "ప్రశాంతమైన శ్వాస",
          "desc":
              "ఈ రోజు ఏదైనా సామాజిక ప్రదేశంలోకి వెళ్లే ముందు 3 సార్లు గట్టిగా శ్వాస తీసుకో.",
        },
        {
          "id": "S18",
          "title": "అక్కడ ఉండటం",
          "desc":
              "రద్దీగా ఉండే ప్రదేశంలో ఫోన్ అస్సలు చూడకుండా 5 నిమిషాల పాటు నిలబడు.",
        },
        {
          "id": "S19",
          "title": "యాదృచ్ఛిక తల ఊపు",
          "desc": "నీతో కళ్ళు కలిపిన అపరిచిత వ్యక్తికి తల ఊపి పలకరించు.",
        },
        {
          "id": "S20",
          "title": "మంచి ముగింపు",
          "desc":
              "షాపు నుండి బయటకు వచ్చేటప్పుడు ఎవరికైనా 'మంచి రోజు అవ్వాలి' అని చెప్పు.",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "ప్రశంసించడం",
          "desc": "తోటి ఉద్యోగి లేదా క్లాస్‌మేట్‌ను మనస్ఫూర్తిగా ప్రశంసించు.",
        },
        {
          "id": "SP2",
          "title": "ప్రశ్న అడగడం",
          "desc": "ఒక అపరిచిత వ్యక్తిని సమయం లేదా దారి అడుగు.",
        },
        {
          "id": "SP3",
          "title": "చిన్న సంభాషణ",
          "desc":
              "ఎవరినైనా 'ఈ రోజు ఎలా గడుస్తోంది?' అని అడిగి, వారి సమాధానాన్ని శ్రద్ధగా విను.",
        },
        {
          "id": "SP4",
          "title": "సహాయం కోరడం",
          "desc":
              "ఒక నిర్దిష్ట వస్తువును కనుగొనడంలో సహాయం చేయమని స్టోర్ ఉద్యోగిని అడుగు.",
        },
        {
          "id": "SP5",
          "title": "ఆర్డర్ ఇచ్చేటప్పుడు",
          "desc":
              "డ్రింక్ లేదా ఫుడ్ ఆర్డర్ ఇచ్చి, అక్కడి స్టాఫ్‌ను వారు ఎలా ఉన్నారని పలకరించు.",
        },
        {
          "id": "SP6",
          "title": "పరిచయం చేసుకోవడం",
          "desc":
              "నీ పరిసరాల్లో ఉన్న ఒక కొత్త వ్యక్తికి నిన్ను నువ్వు పరిచయం చేసుకో.",
        },
        {
          "id": "SP7",
          "title": "వాతావరణం మాటలు",
          "desc":
              "లైన్‌లో వేచి ఉన్నప్పుడు పక్కనున్న వారితో వాతావరణం గురించి మాట్లాడు.",
        },
        {
          "id": "SP8",
          "title": "సాధారణ అన్వేషణ",
          "desc": "సహోద్యోగిని 'వీకెండ్‌లో ఏం చేశారు?' అని అడుగు.",
        },
        {
          "id": "SP9",
          "title": "సహాయం అందివ్వడం",
          "desc":
              "ఎవరైనా ఇబ్బంది పడుతున్నట్లు కనిపిస్తే, 'నేను మీకు సహాయం చేయనా?' అని అడుగు.",
        },
        {
          "id": "SP10",
          "title": "అభిప్రాయం",
          "desc":
              "ఒక చిన్న వస్తువును చూపిస్తూ స్నేహితుడిని 'దీని గురించి నువ్వేమనుకుంటున్నావు?' అని అడుగు.",
        },
        {
          "id": "SP11",
          "title": "ధృవీకరణ",
          "desc":
              "ఒక అపరిచిత వ్యక్తితో ఒక వివరాని కన్ఫర్మ్ చేసుకో (ఉదా: 'ఇదేనా కరెక్ట్ లైన్?').",
        },
        {
          "id": "SP12",
          "title": "పరిసరాల గురించి",
          "desc":
              "చుట్టూ ఉన్న వాతావరణం గురించి ఒక చిన్న కామెంట్ చెయ్ (ఉదా: 'ఇక్కడ నిజంగా చాలా రద్దీగా ఉంది').",
        },
        {
          "id": "SP13",
          "title": "చిన్న సాయం",
          "desc":
              "టేబుల్ మీద ఉన్న ఒక వస్తువును (నాప్‌కిన్ లాంటిది) నీ వైపు జరపమని ఎవరినైనా అడుగు.",
        },
        {
          "id": "SP14",
          "title": "మంచి ఫీడ్‌బ్యాక్",
          "desc": "బయలుదేరే ముందు వెయిటర్‌తో ఫుడ్ చాలా అద్భుతంగా ఉందని చెప్పు.",
        },
        {
          "id": "SP15",
          "title": "సాధారణ పలకరింపు",
          "desc":
              "ఒక నెల నుండి మాట్లాడని వ్యక్తికి 'ఎలా ఉన్నావు?' అని ఒక మెసేజ్ పంపు.",
        },
        {
          "id": "SP16",
          "title": "ఓపెన్ క్వశ్చన్",
          "desc":
              "ఎవరినైనా 'ఈ సిటీలో మీకు బాగా నచ్చిన సందర్శనా స్థలం ఏది?' అని అడుగు.",
        },
        {
          "id": "SP17",
          "title": "అతి చిన్న రిస్క్",
          "desc":
              "దగ్గరలో వాష్‌రూమ్ ఎక్కడ ఉందో తెలుసా అని ఒక అపరిచిత వ్యక్తిని అడుగు.",
        },
        {
          "id": "SP18",
          "title": "వస్తువుల ప్రశంస",
          "desc": "ఎవరికైనా వారి షూస్/బ్యాగ్/యాక్సెసరీ బాగుందని చెప్పు.",
        },
        {
          "id": "SP19",
          "title": "మర్యాదపూర్వక నిరీక్షణ",
          "desc":
              "ఎవరికైనా సమాధానం ఇచ్చే ముందు వారి మాట పూర్తిగా ముగిసే వరకు ఓపికగా వేచి ఉండు.",
        },
        {
          "id": "SP20",
          "title": "స్నేహపూర్వక వీడ్కోలు",
          "desc":
              "ఇప్పుడే నీతో చిన్నగా మాట్లాడిన వ్యక్తికి చేయి ఊపి 'బై' చెప్పు.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "అభిప్రాయాల అన్వేషి",
          "desc":
              "ఒక పుస్తకం, సినిమా లేదా పాటపై ఎవరినైనా వారి అభిప్రాయాన్ని అడుగు.",
        },
        {
          "id": "L2",
          "title": "వివరాలు",
          "desc":
              "ఒకరు తమ గురించి ఏదైనా చెప్పినప్పుడు దానికి సంబంధించిన మరొక ప్రశ్న అడుగు.",
        },
        {
          "id": "L3",
          "title": "సిఫార్సు",
          "desc":
              "దగ్గరలో తినడానికి మంచి హోటల్ ఏదైనా ఉంటే చెప్పమని ఒక అపరిచిత వ్యక్తిని అడుగు.",
        },
        {
          "id": "L4",
          "title": "ఉమ్మడి కనెక్ట్",
          "desc":
              "ఒకరితో ఒక ఉమ్మడి ఆసక్తిని కనుగొని, దాని గురించి 2 నిమిషాల పాటు మాట్లాడు.",
        },
        {
          "id": "L5",
          "title": "సహాయ హస్తం",
          "desc":
              "ఒక చిన్న పనిలో (బ్యాగ్ మోయడం లాంటిది) సహాయం చేయడానికి ముందుండు.",
        },
        {
          "id": "L6",
          "title": "సామాజಿಕ పరిశీలన",
          "desc":
              "మీ ఇద్దరి చుట్టూ జరుగుతున్న ఒక సంఘటన ఆధారంగా సంభాషణను ప్రారంభించు.",
        },
        {
          "id": "L7",
          "title": "ఓపెన్ క్వశ్చన్",
          "desc":
              "ఎవరినైనా 'మీరు ఈ వృత్తిలోకి లేదా పనిలోకి ఎలా వచ్చారు?' అని అడుగు.",
        },
        {
          "id": "L8",
          "title": "చురుకైన శ్రోత",
          "desc":
              "ఎవరి మాటలనైనా మధ్యలో ఆపకుండా 3 నిమిషాల పాటు విను, ఆపై వారు చెప్పినదాన్ని క్లుప్తంగా చెప్పు.",
        },
        {
          "id": "L9",
          "title": "ఉమ్మడి నవ్వు",
          "desc":
              "ఒక చిన్న సమూహానికి ఒక చిన్న, హాస్యభరితమైన కథ లేదా జోక్ చెప్పు.",
        },
        {
          "id": "L10",
          "title": "కుతూహలం",
          "desc":
              "ఎవరినైనా వారు ఏ ఊరు అని, ఆ ఊరిలో వారికి ఏం నచ్చుతుందో అడుగు.",
        },
        {
          "id": "L11",
          "title": "నిజమైన ఆసక్తి",
          "desc": "సహోద్యోగిని పనికి వెలుపల ఉన్న వారి హాబీల గురించి అడుగు.",
        },
        {
          "id": "L12",
          "title": "మృదువైన సలహా",
          "desc":
              "నువ్వు నిపుణుడివి అయిన ఒక విషయంపై ఎవరికైనా ఉపయోగకరమైన సలహా ఇవ్వు.",
        },
        {
          "id": "L13",
          "title": "గ్రూప్‌లో మద్దతు",
          "desc": "ఒక చిన్న గ్రూప్ చర్చలో ఒకరి అభిప్రాయానికి మద్దతుగా తల ఊపు.",
        },
        {
          "id": "L14",
          "title": "సాధారణ ఆహ్వానం",
          "desc":
              "ఎవరినైనా 'మధ్యాహ్నం భోజనానికి మాతో కలిసి వస్తారా?' అని అడుగు.",
        },
        {
          "id": "L15",
          "title": "నిజాయితీ గల రిఫ్లెక్షన్",
          "desc":
              "ఎవరికైనా 'నువ్వు X చేసినప్పుడు నాకు చాలా సంతోషంగా అనిపించింది' అని చెప్పి ఎందుకో వివరించు.",
        },
        {
          "id": "L16",
          "title": "కుతూహలాల గ్యాప్",
          "desc":
              "ఎవరినైనా 'నేను ఎప్పుడూ ఆలోచిస్తాను, ఎక్స్ (X) నిజంగా ఎలా పనిచేస్తుంది?' అని అడుగు.",
        },
        {
          "id": "L17",
          "title": "చిన్న గ్రూప్ లీడ్",
          "desc":
              "ఒక గ్రూప్‌లో 2 లేదా 3 మంది సమాధానం చెప్పాల్సిన ఒక ప్రశ్న అడుగు.",
        },
        {
          "id": "L18",
          "title": "నిజమైన ప్రశంస",
          "desc":
              "ఒకరి వ్యక్తిత్వ లక్షణాన్ని ప్రశంసించు (ఉదా: 'నువ్వు చాలా బాగా వింటావు').",
        },
        {
          "id": "L19",
          "title": "ఉమ్మడి అనుభవం",
          "desc": "సంభాషణ సమయంలో 'నేను కూడా ఆ పరిస్థితిలో ఉన్నాను' అని చెప్పు.",
        },
        {
          "id": "L20",
          "title": "అర్థవంతమైన విరామం",
          "desc":
              "సంభాషణలో నిశ్శబ్ద క్షణం ఏర్పడితే దాన్ని వెంటనే మాటలతో నింపడానికి ప్రయత్నించకుండా సహజంగా ఉండు.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "ధైర్యవంతమైన ప్రారంభం",
          "desc": "నీకు అంతగా తెలియని ఒకరితో సంభాషణను ప్రారంభించు.",
        },
        {
          "id": "ST2",
          "title": "నిజాయితీ గల భాగస్వామ్యం",
          "desc":
              "ఒక గ్రూప్ సెట్టింగ్‌లో ఒక చిన్న వ్యక్తిగత కథ లేదా అభిప్రాయాన్ని పంచుకో.",
        },
        {
          "id": "ST3",
          "title": "చర్చ",
          "desc":
              "ఒకరి అభిప్రాయంతో మర్యాదగా విభేదించి, దానికి గల కారణాన్ని వివరించు.",
        },
        {
          "id": "ST4",
          "title": "గ్రూప్‌లో చేరడం",
          "desc":
              "జరుగుతున్న గ్రూప్ సంభాషణలో చేరి, ఒక ఆలోచనాత్మకమైన వాక్యాన్ని జోడించు.",
        },
        {
          "id": "ST5",
          "title": "టాపిక్ ప్రారంభం",
          "desc":
              "ఒక సామాజిక సమూహంలో చర్చ కోసం ఒక కొత్త టాపిక్‌ను పరిచయం చేయి.",
        },
        {
          "id": "ST6",
          "title": "సభలో ప్రశ్న",
          "desc":
              "ఒక బహిరంగ సమావేశంలో లేదా క్లాస్‌రూమ్ వాతావరణంలో ఒక ప్రశ్న అడుగు.",
        },
        {
          "id": "ST7",
          "title": "ధైర్యవంతమైన అభ్యర్థన",
          "desc":
              "ఒక కేఫ్ లేదా పార్క్‌లో ఒక అపరిచిత వ్యక్తిని వారి పక్కన కూర్చోవచ్చా అని అడుగు.",
        },
        {
          "id": "ST8",
          "title": "సంభాషణ వంతెన",
          "desc":
              "ఒకరినొకరు తెలియని ఇద్దరిని పరిచయం చేసి, వారి మధ్య ఒక ఉమ్మడి అంశాన్ని కనుగొను.",
        },
        {
          "id": "ST9",
          "title": "ఖచ్చితమైన అవసరం",
          "desc":
              "నిన్ను ఇబ్బంది పెట్టే ఒక విషయాన్ని ఆపమని లేదా ఒకరిని పక్కకు జరగమని మర్యాదగా చెప్పు.",
        },
        {
          "id": "ST10",
          "title": "కథకుడు",
          "desc":
              "3 లేదా అంతకంటే ఎక్కువ మంది ఉన్న గ్రూప్‌కు ఒక కథను చెప్పడంలో నాయకత్వం వహించు.",
        },
        {
          "id": "ST11",
          "title": "ఓపెన్ ఛాలెంజ్",
          "desc":
              "ఒక గ్రూప్‌లోని సాధారణ అభిప్రాయాన్ని ఒక స్నేహపూర్వక మరియు గౌరవప్రదమైన మార్గంలో ఛాలెంజ్ చేయి.",
        },
        {
          "id": "ST12",
          "title": "సామాజిక చొరవ",
          "desc":
              "ఒక గదిలోకి ప్రవేశించేటప్పుడు అందరికీ మొదట 'నమస్తే' చెప్పే వ్యక్తివి అవ్వు.",
        },
        {
          "id": "ST13",
          "title": "సహానుభూతితో వినడం",
          "desc":
              "ఎవరైనా తన మనసులోని బాధను లేదా కోపాన్ని చెప్తున్నప్పుడు దాన్ని విని, మద్దతుగా సమాధానం ఇవ్వు.",
        },
        {
          "id": "ST14",
          "title": "బహిరంగ ప్రదర్శన",
          "desc":
              "ఒక సామాజిక సమావేశంలో నీకు ఇష్టమైన ఒక టాపిక్ గురించి 1-2 నిమిషాలు మాట్లాడు.",
        },
        {
          "id": "ST15",
          "title": "బలహీనతను ఒప్పుకోవడం",
          "desc":
              "ఒక గ్రూప్ ముందు నువ్వు ఏదో ఒక విషయంలో నర్వస్‌గా ఉన్నావని ఒప్పుకొని, దాని గురించి అందరూ కలిసి నవ్వుకోండి.",
        },
        {
          "id": "ST16",
          "title": "సరిహద్దులు గీయడం",
          "desc":
              "ఎటువంటి సుదీర్ఘ వివరణ ఇవ్వకుండా, నువ్వు వెళ్లకూడదనుకున్న ఆహ్వానాన్ని మర్యాదగా తిరస్కరించు.",
        },
        {
          "id": "ST17",
          "title": "యాక్టివ్ మధ్యవర్తి",
          "desc":
              "ఒక అభిప్రాయ భేదంలో ఇద్దరు వ్యక్తులను ఒక మధ్యస్థ నిర్ణయానికి తీసుకురావడానికి సహాయపడు.",
        },
        {
          "id": "ST18",
          "title": "బహిరంగ ప్రశంస",
          "desc":
              "ఒక గ్రూప్‌లో ఒకరి ప్రయత్నాన్ని లేదా సాధించిన విజయాన్ని బహిరంగంగా ప్రశంసించు.",
        },
        {
          "id": "ST19",
          "title": "నేరడి పద్ధతి",
          "desc":
              "నీకు అవసరమైన ఒక సహాయం లేదా సలహా కోసం ఒకరిని నేరుగా అభ్యర్థించు.",
        },
        {
          "id": "ST20",
          "title": "సంభాషణ మలుపు",
          "desc":
              "ఒక సంభాషణను ఒక బోరింగ్ టాపిక్ నుండి ఆసక్తికరమైన టాపిక్‌కి మెల్లగా మార్చు.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "బహుమతి",
          "desc":
              "ఎవరికైనా ఒక చిన్న గిఫ్ట్ లేదా తినే వస్తువు ఇచ్చి 'నువ్వు ఇది ఇష్టపడతావని అనుకున్నాను' అని చెప్పు.",
        },
        {
          "id": "B2",
          "title": "ధైర్యవంతమైన నాయకత్వం",
          "desc":
              "ఒక చిన్న గ్రూప్ వ్యక్తులకు ఒక ప్లాన్ లేదా ఏదైనా స్థలాన్ని సందర్శించే ఆలోచనను ప్రతిపాదించు.",
        },
        {
          "id": "B3",
          "title": "విలువ వ్యక్తపరచడం",
          "desc":
              "నీ జీవితంలో వారి ఉనికిని నువ్వు ఎందుకు విలువైనదిగా భావిస్తున్నావో ఒకరికి ప్రత్యేకంగా చెప్పు.",
        },
        {
          "id": "B4",
          "title": "సామాజిక నిర్వాహకుడు",
          "desc":
              "కొంతమంది వ్యక్తుల కోసం ఒక చిన్న మీటింగ్ లేదా ఒక కాఫీ డేట్‌ను ఏర్పాటు చేయి.",
        },
        {
          "id": "B5",
          "title": "లోతైన సంభాషణ",
          "desc":
              "ఒకరితో 15 నిమిషాల కంటే ఎక్కువ సమయం పాటు లోతైన మరియు అర్థవంతమైన సంభాషణను సాగించు.",
        },
        {
          "id": "B6",
          "title": "ఆత్మవిశ్వాస శిఖరం",
          "desc":
              "నిన్ను చూసినప్పుడు కాస్త భయం లేదా వెనకడుగు వేసేలా చేసే ఒకరితో సంభాషణను ప్రారంభించు.",
        },
        {
          "id": "B7",
          "title": "బహిరంగ శుభాకాంక్ష",
          "desc":
              "ఒక గ్రూప్‌లో ఒక నిర్దిష్ట వ్యక్తి కోసం ఒక చిన్న, సానుకూల ప్రశంసల వాక్యాన్ని చెప్పు లేదా వారి ప్రయత్నాన్ని స్వాగతించు.",
        },
        {
          "id": "B8",
          "title": "సరిహద్దులు గీసేవాడు",
          "desc":
              "ఎటువంటి సుదీర్gh వివరణ లేకుండా, ఒక అభ్యర్థనకు గట్టిగా కానీ కనత్వంతో 'లేదు' అని చెప్పు.",
        },
        {
          "id": "B9",
          "title": "నేరుగా అభ్యర్థన",
          "desc":
              "నువ్వు ఇష్టపడే లేదా గౌరవించే ఒకరితో 10 నిమిషాల మాటలు లేదా మార్గదర్శకత్వం కోసం అభ్యర్థన పెట్టు.",
        },
        {
          "id": "B10",
          "title": "భావోద్వేగ ప్రయత్నం",
          "desc":
              "నీ స్నేహితుడు ఒకరితో భావాలు లేదా మానసిక ఆరోగ్యం గురించి లోతైన చర్చను ప్రారంభించు.",
        },
        {
          "id": "B11",
          "title": "సామాజిక మధ్యవర్తి",
          "desc":
              "ప్రశాంతమైన సంభాషణ ద్వారా ఇద్దరు వ్యక్తుల మధ్య ఒక చిన్న వివాదాన్ని పరిష్కరించడానికి సహాయపడు.",
        },
        {
          "id": "B12",
          "title": "ధైర్యవంతమైన ప్రశంస",
          "desc":
              "పూర్తిగా అపరిచిత వ్యక్తి అయిన ఒకరితో నువ్వు నిజంగానే వారిలో ప్రశంసించే ఒక విషయాన్ని చెప్పు.",
        },
        {
          "id": "B13",
          "title": "నెట్‌వర్కింగ్ అడుగు",
          "desc":
              "నీ రంగానికి చెందిన ఒక నిపుణుడు లేదా అర్హత గల వ్యక్తితో నిన్ను నువ్వు పరిచయం చేసుకొని సలహా అడుగు.",
        },
        {
          "id": "B14",
          "title": "ధైర్యవంతమైన నిజం",
          "desc":
              "ఒకరితో చెప్పడానికి కష్టమైన కానీ బంధానికి ఉపయోగపడే ఒక నిజాన్ని చెప్పు.",
        },
        {
          "id": "B15",
          "title": "పూర్తి వికాసం",
          "desc":
              "ఒక చిన్న సామాజిక కార్యక్రమాన్ని ఏర్పాటు చేసి, అక్కడికి వచ్చే ప్రతి అతిథి సౌకర్యంగా ఉండేలా చూడు.",
        },
        {
          "id": "B16",
          "title": "బహిరంగ స్పీకర్",
          "desc":
              "ఒక మీటింగ్ లేదా ఈవెంట్ యొక్క ఒక చిన్న భాగాన్ని లీడ్ చేయడానికి లేదా మాట్లాడటానికి నువ్వుగా ముందు రా.",
        },
        {
          "id": "B17",
          "title": "అనుభవాల గైడ్",
          "desc":
              "మరొకరిని ప్రోత్సహించడానికి నీ కష్టమైన పరిస్థితి లేదా నువ్వు దాటిన ఓటమి అనుభవాన్ని పంచుకో.",
        },
        {
          "id": "B18",
          "title": "ధైర్యవంతమైన క్షమాపణ",
          "desc":
              "గతంలోని ఒక తప్పు కోసం క్షమాపణ అడగడానికి నువ్వుగా సంభాషణను ప్రారంభించు, అది ఎంత పాతదైనా సరే.",
        },
        {
          "id": "B19",
          "title": "మెంటర్",
          "desc":
              "నీకంటే అనుభవంలో తక్కువ ఉన్న ఒక వ్యక్తికి ఒక స్కిల్ లేదా పనిలో సహాయం చేయడానికి ముందుండు.",
        },
        {
          "id": "B20",
          "title": "సామాజిక ఆర్కిటెక్ట్",
          "desc":
              "స్నేహితుల గ్రూప్ కోసం ఒక కొత్త సామాజical సాంప్రదాయాన్ని లేదా తరచుగా జరిగే ఒక మీటప్‌ను ప్రారంభించు.",
        },
      ],
    },
    'kn': {
      "Seedling": [
        {
          "id": "S1",
          "title": "ಮೊದಲ ಹೆಜ್ಜೆ",
          "desc":
              "ಇಂದು ಒಬ್ಬರೊಂದಿಗೆ ಕಣ್ಣುಗಳನ್ನು ಬೆರೆಸಿ, ನೋಡಿ ಸಣ್ಣದಾಗಿ ಮುಗುಳ್ನಕ್ಕು.",
        },
        {
          "id": "S2",
          "title": "ಒಂದು ಸರಳ ಹಲೋ",
          "desc": "ನೆರೆಹೊರೆಯವರೊಂದಿಗೆ 'ಶುಭೋದಯ' ಅಥವಾ 'ನಮಸ್ಕಾರ' ಎಂದು ಹೇಳು.",
        },
        {
          "id": "S3",
          "title": "ಧನ್ಯವಾದಗಳು",
          "desc": "ಅಂಗಡಿಯವನಿಗೆ ಸ್ಪಷ್ಟವಾಗಿ 'ಧನ್ಯವಾದಗಳು' ಎಂದು ಹೇಳು.",
        },
        {
          "id": "S4",
          "title": "ಪರಿಶೀಲನೆ",
          "desc":
              "ಒಬ್ಬ ಅಪರಿಚಿತ ವ್ಯಕ್ತಿಯಲ್ಲಿ ಯಾವುದಾದರೂ ಒಳ್ಳೆಯ ವಿಷಯವನ್ನು ಗಮನಿಸಿ ಮುಗುಳ್ನಕ್ಕು.",
        },
        {
          "id": "S5",
          "title": "ನಿಶ್ಯಬ್ದ ಹಸ್ತಚಾಲನ",
          "desc": "ದೂರದಿಂದ ನೀನು ಗುರುತಿಸಿದ ಯಾರಿಗಾದರೂ ಸಣ್ಣದಾಗಿ ಕೈ ಬೀಸು.",
        },
        {
          "id": "S6",
          "title": "ಬಾಗಿಲು ಹಿಡಿಯುವುದು",
          "desc": "ನಿನ್ನ ಹಿಂದೆ ಬರುತ್ತಿರುವವರಿಗಾಗಿ ಬಾಗಿಲು ತೆರೆದು ಹಿಡಿದುಕೋ.",
        },
        {
          "id": "S7",
          "title": "ತಲೆ ಆಡಿಸುವುದು",
          "desc":
              "ಪಕ್ಕದಿಂದ ಹೋಗುವಾಗ ಸಹೋದ್ಯೋಗಿಗೆ ಸ್ನೇಹಪೂರ್ವಕವಾಗಿ ತಲೆ ಆಡಿಸಿ ನಮಸ್ಕರಿಸು.",
        },
        {
          "id": "S8",
          "title": "ಕನ್ನಡಿಯ ಪ್ರಾಕ್ಟೀಸ್",
          "desc":
              "ಕನ್ನಡಿಯ ಮುಂದೆ ನಿಂತು 1 ನಿಮಿಷಗಳ ಕಾಲ ನಿನ್ನ 'ಆತ್ಮವಿಶ್ವಾಸದ ಮುಗುಳ್ನಗೆಯನ್ನು' ಪ್ರಾಕ್ಟೀಸ್ ಮಾಡು.",
        },
        {
          "id": "S9",
          "title": "ಚಿಕ್ಕ ನೋಟ",
          "desc":
              "ಯಾರನ್ನಾದರೂ 2 ಸೆಕೆಂಡುಗಳ ಕಾಲ ನೋಡಿ, ಮುಗುಳ್ನಕ್ಕು ನಂತರ ದೃಷ್ಟಿ ತಿರುಗಿಸಿಕೋ.",
        },
        {
          "id": "S10",
          "title": "ನಿಶ್ಯಬ್ದ ಪ್ರಶಂಸೆ",
          "desc":
              "ಯಾರದ್ದಾದರೂ ಸೋಶಿಯಲ್ ಮೀಡಿಯಾ ಪೋಸ್ಟ್ ಅಡಿಯಲ್ಲಿ ಒಂದು ಒಳ್ಳೆಯ ಕಾಮೆಂಟ್ ಬರೆ.",
        },
        {
          "id": "S11",
          "title": "ಸ್ಥಳವನ್ನು ಹಂಚಿಕೊಳ್ಳುವುದು",
          "desc":
              "ಪಬ್ಲಿಕ್ ಏರಿಯಾದಲ್ಲಿ ಯಾರಿಗಾದರೂ ಪಕ್ಕದಲ್ಲಿ ಕುಳಿತುಕೊ ಮತ್ತು ತಕ್ಷಣ ದೃಷ್ಟಿ ತಿರುಗಿಸದಂತೆ ಇರು.",
        },
        {
          "id": "S12",
          "title": "ಸರಳ ಮರ್ಯಾದೆ",
          "desc":
              "ಕಾರಿಡಾರ್‌ನಲ್ಲಿ ಯಾರನ್ನಾದರೂ ದಾಟಿ ಹೋಗುವಾಗ ಮರ್ಯಾದೆಯಾಗಿ 'ಎಕ್ಸ್‌ಕ್ಯೂಸ್ ಮೀ' ಎಂದು ಹೇಳು.",
        },
        {
          "id": "S13",
          "title": "ಆತ್ಮೀಯ ಪಲಕರಿಕೆ",
          "desc": "ಡೆಲಿವರಿ ಬಾಯ್ ಅಥವಾ ಕೊರಿಯರ್ ವ್ಯಕ್ತಿಗೆ 'ನಮಸ್ಕಾರ' ಎಂದು ಹೇಳು.",
        },
        {
          "id": "S14",
          "title": "ಚಿಕ್ಕ ಸೈಗೆ",
          "desc":
              "ಒಂದು ಸಣ್ಣ ಮಗುವಿಗೆ ಅಥವಾ (ಯಜಮಾನನ ಅನುಮತಿಯೊಂದಿಗೆ) ಸಾಕುಪ್ರಾಣಿಗೆ ಕೈ ಬೀሱ.",
        },
        {
          "id": "S15",
          "title": "ಮೃದುವಾದ ಮುಗುಳ್ನಗೆ",
          "desc": "ಇಂದು ಮೂವರು ಬೇರೆ ಬೇರೆ ವ್ಯಕ್ತಿಗಳನ್ನು ನೋಡಿ ಮುಗುಳ್ನಕ್ಕು.",
        },
        {
          "id": "S16",
          "title": "ಐ ಕಾಂಟ್ಯಾಕ್ಟ್ ಸವಾಲು",
          "desc":
              "ಕ್ಯಾಷಿಯರ್ ಮೊದಲು ದೃಷ್ಟಿ ತಿರುಗಿಸುವವರೆಗೂ ಅವರೊಂದಿಗೆ ಕಣ್ಣುಗಳನ್ನು ಬೆರೆಸಿ ಇಡು.",
        },
        {
          "id": "S17",
          "title": "ಪ್ರಶಾಂತವಾದ ಶ್ವಾಸ",
          "desc":
              "ಇಂದು ಯಾವುದೇ ಸಾಮಾಜಿಕ ಪ್ರದೇಶಕ್ಕೆ ಹೋಗುವ ಮುನ್ನ 3 ಬಾರಿ ಜೋರಾಗಿ ಶ್ವಾಸ ತೆಗೆದುಕೋ.",
        },
        {
          "id": "S18",
          "title": "ಅಲ್ಲಿ ಇರುವುದು",
          "desc":
              "ಜನನಿಬಿಡ ಪ್ರದೇಶದಲ್ಲಿ ಫೋನ್ ಅಸ್ಸಲು ನೋಡದಂತೆ 5 ನಿಮಿಷಗಳ ಕಾಲ ನಿಲ್ಲಬೇಕು.",
        },
        {
          "id": "S19",
          "title": "ಯಾದೃಚ್ಛಿಕ ತಲೆ ಆಡಿಸುವಿಕೆ",
          "desc":
              "ನಿನ್ನೊಂದಿಗೆ ಕಣ್ಣುಗಳನ್ನು ಬೆರೆಸಿದ ಅಪರಿಚಿತ ವ್ಯಕ್ತಿಗೆ ತಲೆ ಆಡಿಸಿ ಪಲಕರಿಸು.",
        },
        {
          "id": "S20",
          "title": "ಒಳ್ಳೆಯ ಮುಕ್ತಾಯ",
          "desc":
              "ಶಾಪ್‌ನಿಂದ ಹೊರಗೆ ಬರುವಾಗ ಯಾರಿಗಾದರೂ 'ಒಳ್ಳೆಯ ದಿನವಾಗಲಿ' ಎಂದು ಹೇಳು.",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "ಪ್ರಶಂಸಿಸುವುದು",
          "desc":
              "ಸಹೋದ್ಯೋಗಿ ಅಥವಾ ಕ್ಲಾಸ್‌ಮೇಟ್ ಅನ್ನು ಮನಸ್ಸುಪೂರ್ವಕವಾಗಿ ಪ್ರಶಂಸಿಸು.",
        },
        {
          "id": "SP2",
          "title": "ಪ್ರಶ್ನೆ ಕೇಳುವುದು",
          "desc": "ಒಬ್ಬ ಅಪರಿಚಿತ ವ್ಯಕ್ತಿಯನ್ನು ಸಮಯ ಅಥವಾ ದಾರಿ ಕೇಳು.",
        },
        {
          "id": "SP3",
          "title": "ಚಿಕ್ಕ ಸಂಭಾಷಣೆ",
          "desc":
              "ಯಾರನ್ನಾದರೂ 'ಇಂದು ದಿನ ಹೇಗೆ ನಡೆಯುತ್ತಿದೆ?' ಎಂದು ಕೇಳಿ, ಅವರ ಉತ್ತರವನ್ನು ಶ್ರದ್ಧೆಯಿಂದ ಕೇಳು.",
        },
        {
          "id": "SP4",
          "title": "ಸಹಾಯ ಕೋರುವುದು",
          "desc":
              "ಒಂದು ನಿರ್ದಿಷ್ಟ ವಸ್ತುವನ್ನು ಹುಡುಕುವಲ್ಲಿ ಸಹಾಯ ಮಾಡು ಎಂದು ಸ್ಟೋರ್ ಉದ್ಯೋಗಿಯನ್ನು ಕೇಳು.",
        },
        {
          "id": "SP5",
          "title": "ಆರ್ಡರ್ ಕೊಡುವಾಗ",
          "desc":
              "ಡ್ರಿಂಕ್ ಅಥವಾ ಫುಡ್ ಆರ್ಡರ್ ಕೊಟ್ಟು, ಅಕ್ಕಪಕ್ಕದ ಸ್ಟಾಫ್ ಅನ್ನು ಅವರು ಹೇಗಿದ್ದಾರೆ ಎಂದು ಪಲಕರಿಸು.",
        },
        {
          "id": "SP6",
          "title": "ಪರಿಚಯ ಮಾಡಿಕೊಳ್ಳುವುದು",
          "desc":
              "ನಿನ್ನ ಪರಿಸರದಲ್ಲಿ ಇರುವ ಒಂದು ಹೊಸ ವ್ಯಕ್ತಿಗೆ ನಿನ್ನನ್ನು ನೀನು ಪರಿಚಯ ಮಾಡಿಕೋ.",
        },
        {
          "id": "SP7",
          "title": "ಹವಾಮಾನದ ಮಾತುಗಳು",
          "desc":
              "ಲೈನ್‌ನಲ್ಲಿ ಕಾಯುತ್ತಿರುವಾಗ ಪಕ್ಕದಲ್ಲಿರುವವರೊಂದಿಗೆ ಹವಾಮಾನದ ಬಗ್ಗೆ ಮಾತನಾಡು.",
        },
        {
          "id": "SP8",
          "title": "ಸರಳ ಅನ್ವೇಷಣೆ",
          "desc": "ಸಹೋದ್ಯೋಗಿಯನ್ನು 'ವೀಕೆಂಡ್‌ನಲ್ಲಿ ಏನು ಮಾಡಿದ್ರಿ?' ಎಂದು ಕೇಳು.",
        },
        {
          "id": "SP9",
          "title": "ಸಹಾಯ ಒದಗಿಸುವುದು",
          "desc":
              "ಯಾರಾದರೂ ಇಬ್ಬಂದಿ ಪಡುತ್ತಿರುವುದು ಕಂಡರೆ, 'ನಾನು ನಿಮಗೆ ಸಹಾಯ ಮಾಡಲೇ?' ಎಂದು ಕೇಳು.",
        },
        {
          "id": "SP10",
          "title": "ಅಭಿಪ್ರಾಯ",
          "desc":
              "ಒಂದು ಚಿಕ್ಕ ವಸ್ತುವನ್ನು ತೋರಿಸುತ್ತಾ ಸ್ನೇಹಿತನನ್ನು 'ಇದರ ಬಗ್ಗೆ ನೀನೇನಂತೀಯಾ?' ಎಂದು ಕೇಳು.",
        },
        {
          "id": "SP11",
          "title": "ದೃಢೀಕರಣ",
          "desc":
              "ಒಬ್ಬ ಅಪриಚಿತ ವ್ಯಕ್ತಿಯೊಂದಿಗೆ ಒಂದು ವಿವರವನ್ನು ಕನ್ಫರ್ಮ್ ಮಾಡಿಕೋ (ಉದಾ: 'ಇದೇನಾ ಕರೆಕ್ಟ್ ಲೈನ್?').",
        },
        {
          "id": "SP12",
          "title": "ಪರಿಸರದ ಬಗ್ಗೆ",
          "desc":
              "ಸುತ್ತಲೂ ಇರುವ ವಾತಾವರಣದ ಬಗ್ಗೆ ಒಂದು ಚಿಕ್ಕ ಕಾಮೆಂಟ್ ಮಾಡು (ಉದಾ: 'ಇಲ್ಲಿ ನಿಜಕ್ಕೂ ತುಂಬಾ ರಶ್ ಆಗಿದೆ').",
        },
        {
          "id": "SP13",
          "title": "ಸಣ್ಣ ಸಾಯಂಕಾಲ",
          "desc":
              "ಟೇಬಲ್ ಮೇಲೆ ಇರುವ ಒಂದು ವಸ್ತುವನ್ನು (ನಾಪ್ಕಿನ್ ತರಹದ್ದು) ನಿನ್ನ ಕಡೆ ಜರುಗಿಸು ಎಂದು ಯಾರನ್ನಾದರೂ ಕೇಳು.",
        },
        {
          "id": "SP14",
          "title": "ಒಳ್ಳೆಯ ಫೀಡ್‌ಬ್ಯಾಕ್",
          "desc":
              "ಹೊರಡುವ ಮುನ್ನ ವೈಟರ್‌ನೊಂದಿಗೆ ಫುಡ್ ತುಂಬಾ ಅದ್ಭುತವಾಗಿದೆ ಎಂದು ಹೇಳು.",
        },
        {
          "id": "SP15",
          "title": "ಸಾಮಾನ್ಯ ಪಲಕರಿಕೆ",
          "desc":
              "ಒಂದು ತಿಂಗಳಿನಿಂದ ಮಾತನಾಡದ ವ್ಯಕ್ತಿಗೆ 'ಹೇಗಿದ್ದೀಯಾ?' ಎಂದು ಒಂದು ಮೆಸೇಜ್ ಕಳುಹಿಸು.",
        },
        {
          "id": "SP16",
          "title": "ಓಪನ್ ಕ್ವೆಶ್ಚನ್",
          "desc":
              "ಯಾರನ್ನಾದರೂ 'ಈ ಸಿಟಿಯಲ್ಲಿ ನಿಮಗೆ ತುಂಬಾ ಇಷ್ಟವಾದ ಸಂದರ್ಶನ ಸ್ಥಳ ಯಾವುದು?' ಎಂದು ಕೇಳು.",
        },
        {
          "id": "SP17",
          "title": "ಅತಿ ಚಿಕ್ಕ ರಿಸ್ಕ್",
          "desc":
              "ಹತ್ತಿರದಲ್ಲಿ ವಾಶ್‌ರೂಮ್ ಎಲ್ಲಿದೆ ಎಂದು ಗೊತ್ತುಂಟಾ ಎಂದು ಒಬ್ಬ ಅಪриಚಿತ ವ್ಯಕ್ತಿಯನ್ನು ಕೇಳು.",
        },
        {
          "id": "SP18",
          "title": "ವಸ್ತುಗಳ ಪ್ರಶಂಸೆ",
          "desc": "ಯಾರಿಗಾದರೂ ಅವರ ಶೂಸ್/ಬ್ಯಾಗ್/ಆಕ್ಸೆಸರಿ ಚೆನ್ನಾಗಿದೆ ಎಂದು ಹೇಳು.",
        },
        {
          "id": "SP19",
          "title": "ಮರ್ಯಾದಾಪೂರ್ವಕ ನಿರೀಕ್ಷಣೆ",
          "desc":
              "ಯಾರಿಗಾದರೂ ಸಮಾಧಾನ ಕೊಡುವ ಮುನ್ನ ಅವರ ಮಾತು ಪೂರ್ತಿಯಾಗಿ ಮುಗಿಯುವವರೆಗೂ ಓಪಿಕೆಯಿಂದ ಕಾಯಿ.",
        },
        {
          "id": "SP20",
          "title": "ಸ್ನೇಹಪೂರ್ವಕ ವೀಳ್ಕೊಡುಗೆ",
          "desc":
              "ಇಪ್ಪತ್ತೇ ನಿನ್ನೊಂದಿಗೆ ಚಿಕ್ಕದಾಗಿ ಮಾತನಾಡಿದ ವ್ಯಕ್ತಿಗೆ ಕೈ ಬೀಸಿ 'ಬೈ' ಹೇಳು.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "ಅಭಿಪ್ರಾಯಗಳ ಅನ್ವೇಷಿ",
          "desc":
              "ಒಂದು ಪುಸ್ತಕ, ಸಿನಿಮಾ ಅಥವಾ ಹಾಡಿನ ಮೇಲೆ ಯಾರನ್ನಾದರೂ ಅವರ ಅಭಿಪ್ರಾಯವನ್ನು ಕೇಳು.",
        },
        {
          "id": "L2",
          "title": "ವಿವರಗಳು",
          "desc":
              "ಒಬ್ಬರು ತಮ್ಮ ಬಗ್ಗೆ ಏನಾದರೂ ಹೇಳಿದಾಗ ಅದಕ್ಕೆ ಸಂಬಂಧಿಸಿದ ಮತ್ತೊಂದು ಪ್ರಶ್ನೆ ಕೇಳು.",
        },
        {
          "id": "L3",
          "title": "ಸಿಫಾರಸು",
          "desc":
              "ಹತ್ತಿರದಲ್ಲಿ ಊಟ ಮಾಡಲು ಒಳ್ಳೆಯ ಹೋಟೆಲ್ ಯಾವುದಾದರೂ ಇದ್ದರೆ ಹೇಳಿ ಎಂದು ಒಬ್ಬ ಅಪриಚಿತ ವ್ಯಕ್ತಿಯನ್ನು ಕೇಳು.",
        },
        {
          "id": "L4",
          "title": "ಉಮ್ಮಡಿ ಕನೆಕ್ಟ್",
          "desc":
              "ಒಬ್ಬರೊಂದಿಗೆ ಒಂದು ಉಮ್ಮಡಿ ಆಸಕ್ತಿಯನ್ನು ಕಂಡುಹಿಡಿದು, ಅದರ ಬಗ್ಗೆ 2 ನಿಮಿಷಗಳ ಕಾಲ ಮಾತನಾಡು.",
        },
        {
          "id": "L5",
          "title": "ಸಹಾಯ ಹಸ್ತ",
          "desc":
              "ಒಂದು ಸಣ್ಣ ಕೆಲಸದಲ್ಲಿ (ಬ್ಯಾಗ್ ಹೊರಡುವುದು ತರಹದ್ದು) ಸಹಾಯ ಮಾಡಲು ಮುಂದುವರಿ.",
        },
        {
          "id": "L6",
          "title": "ಸಾಮಾಜಿಕ ಪರಿಶೀಲನೆ",
          "desc":
              "ನಿಮ್ಮಿಬ್ಬರ ಸುತ್ತ ನಡೆಯುತ್ತಿರುವ ಒಂದು ಸಂಘಟನೆ ಆಧಾರದ ಮೇಲೆ ಸಂಭಾಷಣೆಯನ್ನು ಪ್ರಾರಂಭಿಸು.",
        },
        {
          "id": "L7",
          "title": "ಓಪన్ ಕ್ವೆಶ್ಚನ್",
          "desc":
              "ಯಾರನ್ನಾದರೂ 'ನೀವು ಈ ವೃತ್ತಿಗೆ ಅಥವಾ ಕೆಲಸಕ್ಕೆ ಹೇಗೆ ಬಂದಿರಿ?' ಎಂದು ಕೇಳು.",
        },
        {
          "id": "L8",
          "title": "ಚುರುಕಾದ ಶ್ರೋತ",
          "desc":
              "ಯಾರ ಮಾತುಗಳನ್ನಾದರೂ ಮಧ್ಯದಲ್ಲಿ ನಿಲ್ಲಿಸದೆ 3 ನಿಮಿಷಗಳ ಕಾಲ ಕೇಳು, ಆಮೇಲೆ ಅವರು ಹೇಳಿದ್ದನ್ನು ಕ್ಲುಪ್ತವಾಗಿ ಹೇಳು.",
        },
        {
          "id": "L9",
          "title": "ಉಮ್ಮಡಿ ನಗು",
          "desc":
              "ಒಂದು ಸಣ್ಣ ಸಮೂಹಕ್ಕೆ ಒಂದು ಚಿಕ್ಕ, ಹಾಸ್ಯಭರಿತವಾದ ಕಥೆ ಅಥವಾ ಜೋಕ್ ಹೇಳು.",
        },
        {
          "id": "L10",
          "title": "ಕುತೂಹಲ",
          "desc":
              "ಯಾರನ್ನಾದರೂ ಅವರು ಯಾವ ಊರು ಎಂದು, ಆ ಊರಿನಲ್ಲಿ ಅವರಿಗೆ ಏನು ಇಷ್ಟವಾಗುತ್ತದೆ ಎಂದು ಕೇಳು.",
        },
        {
          "id": "L11",
          "title": "ನಿಜವಾದ ಆಸಕ್ತಿ",
          "desc": "ಸಹೋದ್ಯೋಗಿಯನ್ನು ಕೆಲಸಕ್ಕೆ ಹೊರಗಿರುವ ಅವರ ಹಾಬಿಗಳ ಬಗ್ಗೆ ಕೇಳು.",
        },
        {
          "id": "L12",
          "title": "ಮೃದುವಾದ ಸಲಹೆ",
          "desc":
              "ನೀನು ನಿಪುಣನಾಗಿರುವ ಒಂದು ವಿಷಯದ ಮೇಲೆ ಯಾರಿಗಾದರೂ ಉಪಯೋಗಕರವಾದ ಸಲಹೆ ನೀಡು.",
        },
        {
          "id": "L13",
          "title": "ಗ್ರೂಪ್‌ನಲ್ಲಿ ಬೆಂಬಲ",
          "desc":
              "ಒಂದು ಚಿಕ್ಕ ಗ್ರೂಪ್ ಚರ್ಚೆಯಲ್ಲಿ ಒಬ್ಬರ ಅಭಿಪ್ರಾಯಕ್ಕೆ ಬೆಂಬಲವಾಗಿ ತಲೆ ಆಡಿಸು.",
        },
        {
          "id": "L14",
          "title": "ಸಾಧಾರಣ ಆಹ್ವಾನ",
          "desc":
              "ಯಾರನ್ನಾದರೂ 'ಮಧ್ಯಾಹ್ನ ಭೋಜನಕ್ಕೆ ನಮ್ಮೊಂದಿಗೆ ಕೂಡಿ ಬರುತ್ತೀರಾ?' என்று ಕೇಳು.",
        },
        {
          "id": "L15",
          "title": "ನಿಜಾಯಿತಿ ಗಲ ರಿಫ್ಲೆಕ್ಷನ್",
          "desc":
              "ಯಾರಿಗಾದರೂ 'ನೀನು X ಮಾಡಿದಾಗ ನನಗೆ ತುಂಬಾ ಸಂತೋಷವಾಯಿತು' ಎಂದು ಹೇಳಿ ಏನೆಂದು ವಿವರಿಸು.",
        },
        {
          "id": "L16",
          "title": "ಕುತೂಹಲಗಳ ಗ್ಯಾಪ್",
          "desc":
              "ಯಾರನ್ನಾದರೂ 'ನಾನು ಯಾವಾಗಲೂ ಯೋಚಿಸುತ್ತೇನೆ, ಎಕ್ಸ್ (X) ನಿಜಕ್ಕೂ ಹೇಗೆ ಕೆಲಸ ಮಾಡುತ್ತದೆ?' ಎಂದು ಕೇಳು.",
        },
        {
          "id": "L17",
          "title": "ಚಿನ್ನ ಗ್ರೂಪ್ ಲೀಡ್",
          "desc":
              "ಒಂದು ಗ್ರೂಪ್‌ನಲ್ಲಿ 2 ಅಥವಾ 3 ಮಂದಿ ಸಮಾಧಾನ ಹೇಳಬೇಕಾದ ಒಂದು ಪ್ರಶ್ನೆ ಕೇಳು.",
        },
        {
          "id": "L18",
          "title": "ನಿಜವಾದ ಪ್ರಶಂಸೆ",
          "desc":
              "ಒಬ್ಬರ ವ್ಯಕ್ತಿತ್ವ ಲಕ್ಷಣವನ್ನು ಪ್ರಶಂಸಿಸು (ಉದಾ: 'ನೀನು ತುಂಬಾ ಚೆನ್ನಾಗಿ ಕೇಳಿಸಿಕೊಳ್ಳುತ್ತೀಯಾ').",
        },
        {
          "id": "L19",
          "title": "ಉಮ್ಮಡಿ ಅನುಭವ",
          "desc": "ಸಂಭಾಷಣೆ ಸಮಯದಲ್ಲಿ 'ನಾನು ಕೂಡ ಆ ಪರಿಸ್ಥಿತಿಯಲ್ಲಿದ್ದೆ' ಎಂದು ಹೇಳು.",
        },
        {
          "id": "L20",
          "title": "ಅರ್ಥವಂತಾದ ವಿರಾಮ",
          "desc":
              "ಸಂಭाಷಣೆಯಲ್ಲಿ ನಿಶ್ಯಬ್ದ ಕ್ಷಣ ಏರ್ಪಟ್ಟರೆ ಅದನ್ನು ತಕ್ಷಣ ಮಾತುಗಳಿಂದ ತುಂಬಲು ಪ್ರಯತ್ನಿಸದೆ ಸಹಜವಾಗಿ ಇರು.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "ಧೈರ್ಯವಂತಾದ ಪ್ರಾರಂಭ",
          "desc": "ನಿನಗೆ ಅಷ್ಟಾಗಿ ತಿಳಿಯದ ಒಬ್ಬರೊಂದಿಗೆ ಸಂಭಾಷಣೆಯನ್ನು ಪ್ರಾರಂಭಿಸು.",
        },
        {
          "id": "ST2",
          "title": "ನಿಜಾಯಿತಿ ಗಲ ಭಾಗಸ್ವಾಮ್ಯ",
          "desc":
              "ಒಂದು ಗ್ರೂಪ್ ಸೆಟ್ಟಿಂಗ್‌ನಲ್ಲಿ ಒಂದು ಚಿಕ್ಕ ವೈಯಕ್ತಿಕ ಕಥೆ ಅಥವಾ ಅಭಿಪ್ರಾಯವನ್ನು ಹಂಚಿಕೊ.",
        },
        {
          "id": "ST3",
          "title": "ಚರ್ಚೆ",
          "desc":
              "ಒಬ್ಬರ ಅಭಿಪ್ರಾಯದೊಂದಿಗೆ ಮರ್ಯಾದೆಯಾಗಿ ವಿಭೇದಿಸಿ, ಅದಕ್ಕೆ ಗಲ ಕಾರಣವನ್ನು ವಿವರಿಸು.",
        },
        {
          "id": "ST4",
          "title": "ಗ್ರೂಪ್‌ನಲ್ಲಿ ಸೇರುವುದು",
          "desc":
              "ನಡೆಯುತ್ತಿರುವ ಗ್ರೂಪ್ ಸಂಭಾಷಣೆಯಲ್ಲಿ ಸೇರಿ, ಒಂದು ಆಲೋಚನಾತ್ಮಕವಾದ ವಾಕ್ಯವನ್ನು ಜೋಡಿಸು.",
        },
        {
          "id": "ST5",
          "title": "ಟಾಪಿಕ್ ಪ್ರಾರಂಭ",
          "desc":
              "ಒಂದು ಸಾಮಾಜಿಕ ಸಮೂಹದಲ್ಲಿ ಚರ್ಚೆಗಾಗಿ ಒಂದು ಹೊಸ ಟಾಪಿಕ್ ಅನ್ನು ಪರಿಚಯಿಸು.",
        },
        {
          "id": "ST6",
          "title": "ಸಭೆಯಲ್ಲಿ ಪ್ರಶ್ನೆ",
          "desc":
              "ಒಂದು ಬಹಿರಂಗ ಸಭೆಯಲ್ಲಿ ಅಥವಾ ಕ್ಲಾಸ್‌ರೂಮ್ ವಾತಾವರಣದಲ್ಲಿ ಒಂದು ಪ್ರಶ್ನೆ ಕೇಳು.",
        },
        {
          "id": "ST7",
          "title": "ಧೈರ್ಯವಂತಾದ ಅಭ್ಯರ್ಥನೆ",
          "desc":
              "ಒಂದು ಕೆಫೆ ಅಥವಾ ಪಾರ್ಕ್‌ನಲ್ಲಿ ಒಬ್ಬ ಅಪриचित ವ್ಯಕ್ತಿಯನ್ನು ಅವರ ಪಕ್ಕದಲ್ಲಿ ಕುಳಿತುಕೊಳ್ಳಬಹುದೇ ಎಂದು ಕೇಳು.",
        },
        {
          "id": "ST8",
          "title": "ಸಂಭಾಷಣೆ ಸೇತುವೆ",
          "desc":
              "ಒಬ್ಬರಿಗೊಬ್ಬರು ತಿಳಿಯದ ಇಬ್ಬರನ್ನು ಪರಿಚಯಿಸಿ, ಅವರ ಮಧ್ಯೆ ಒಂದು ಉಮ್ಮಡಿ ಅಂಶವನ್ನು ಕಂಡುಹಿಡಿ.",
        },
        {
          "id": "ST9",
          "title": "ಖಚಿತವಾದ ಅಗತ್ಯತೆ",
          "desc":
              "ನಿನ್ನನ್ನು ಇಬ್ಬಂದಿ ಪಡಿಸುವ ಒಂದು ವಿಷಯವನ್ನು ನಿಲ್ಲಿಸು ಎಂದು ಅಥವಾ ಒಬ್ಬರನ್ನು ಪಕ್ಕಕ್ಕೆ ಜರುಗಲು ಮರ್ಯಾದೆಯಾಗಿ ಹೇಳು.",
        },
        {
          "id": "ST10",
          "title": "ಕಥೆಗಾರ",
          "desc":
              "3 ಅಥವಾ ಅದಕ್ಕಿಂತ ಹೆಚ್ಚು ಮಂದಿ ಇರುವ ಗ್ರೂಪ್‌ಗೆ ಒಂದು ಕಥೆಯನ್ನು ಹೇಳುವುದರಲ್ಲಿ ನಾಯಕತ್ವ ವಹಿಸು.",
        },
        {
          "id": "ST11",
          "title": "ಓಪನ್ ಚಾಲೆಂಜ್",
          "desc":
              "ಒಂದು ಗ್ರೂಪ್‌ನಲ್ಲಿನ ಸಾಮಾನ್ಯ ಅಭಿಪ್ರಾಯವನ್ನು ಒಂದು ಸ್ನೇಹಪೂರ್ವಕ ಮತ್ತು ಗೌರವಪ್ರದವಾದ ಮಾರ್ಗದಲ್ಲಿ ಚಾಲೆಂಜ್ ಮಾಡು.",
        },
        {
          "id": "ST12",
          "title": "ಸಾಮಾಜಿಕ ಚೊರವೆ",
          "desc":
              "ಒಂದು ಕೋಣೆಗೆ ಪ್ರವೇಶಿಸುವಾಗ ಎಲ್ಲರಿಗೂ ಮೊದಲು 'ನಮಸ್ಕಾರ' ಹೇಳುವ ವ್ಯಕ್ತಿಯಾಗು.",
        },
        {
          "id": "ST13",
          "title": "ಸಹಾನುಭೂತಿಯಿಂದ ಕೇಳುವುದು",
          "desc":
              "ಯಾರಾದರೂ ತನ್ನ ಮನಸ್ಸಿನ ಬಾದೆಯನ್ನು ಅಥವಾ ಕೋಪವನ್ನು ಹೇಳುತ್ತಿರುವಾಗ ಅದನ್ನು ಕೇಳಿ, ಬೆಂಬಲವಾಗಿ ಸಮಾಧಾನ ನೀಡು.",
        },
        {
          "id": "ST14",
          "title": "ಬಹಿರಂಗ ಪ್ರದರ್ಶನ",
          "desc":
              "ಒಂದು ಸಾಮಾಜಿಕ ಸಭೆಯಲ್ಲಿ ನಿನಗೆ ಇಷ್ಟವಾದ ಒಂದು ಟಾಪಿಕ್ ಬಗ್ಗೆ 1-2 ನಿಮಿಷ ಮಾತನಾಡು.",
        },
        {
          "id": "ST15",
          "title": "ಬಲಹೀನತೆಯನ್ನು ಒಪ್ಪಿಕೊಳ್ಳುವುದು",
          "desc":
              "ಒಂದು ಗ್ರೂಪ್ ಮುಂದೆ ನೀನು ಯಾವುದೋ ಒಂದು ವಿಷಯದಲ್ಲಿ ನರ್ವಸ್ ಆಗಿದ್ದೀಯಾ ಎಂದು ಒಪ್ಪಿಕೊಂಡು, ಅದರ ಬಗ್ಗೆ ಎಲ್ಲರೂ ಕೂಡಿ ನಗು.",
        },
        {
          "id": "ST16",
          "title": "ಸರಿಹದ್ದುಗಳು ಗೀಳುವುದು",
          "desc":
              "ಎಂತಹ ಸುದೀರ್ಘ ವಿವರಣೆ ನೀಡದೆ, ನೀನು ಹೋಗಬಾರದೆಂದುಕೊಂಡ ಆಹ್ವಾನವನ್ನು ಮರ್ಯಾದೆಯಾಗಿ ತಿರಸ್ಕರಿಸು.",
        },
        {
          "id": "ST17",
          "title": "ಯಾಕ್ಟಿವ್ ಮಧ್ಯವರ್ತಿ",
          "desc":
              "ಒಂದು ಅಭಿಪ್ರಾಯ ಭೇದದಲ್ಲಿ ಇಬ್ಬರು ವ್ಯಕ್ತಿಗಳನ್ನು ಒಂದು ಮಧ್ಯಸ್ಥ ನಿರ್ಧಾರಕ್ಕೆ ಕರೆತರಲು ಸಹಾಯ ಮಾಡು.",
        },
        {
          "id": "ST18",
          "title": "ಬಹಿರಂಗ ಪ್ರಶಂಸೆ",
          "desc":
              "ಒಂದು ಗ್ರೂಪ್‌ನಲ್ಲಿ ಒಬ್ಬರ ಪ್ರಯತ್ನವನ್ನು ಅಥವಾ ಸಾಧಿಸಿದ ವಿಜಯವನ್ನು ಬಹಿರಂಗವಾಗಿ ಪ್ರಶಂಸಿಸು.",
        },
        {
          "id": "ST19",
          "title": "ನೇರಡಿ ಪದ್ಧತಿ",
          "desc":
              "ನಿನಗೆ ಅಗತ್ಯವಿರುವ ಒಂದು ಸಹಾಯ ಅಥವಾ ಸಲಹೆಗಾಗಿ ಒಬ್ಬರನ್ನು ನೇರವಾಗಿ ಅಭ್ಯರ್ಥಿಸು.",
        },
        {
          "id": "ST20",
          "title": "ಸಂಭಾಷಣೆ ಮలుಪು",
          "desc":
              "ಒಂದು ಸಂಭಾಷಣೆಯನ್ನು ಒಂದು ಬೋರಿಂಗ್ ಟಾಪಿಕ್‌ನಿಂದ ಆಸಕ್ತಿ ಕರವಾದ ಟಾಪಿಕ್‌ಗೆ ಮೆಲ್ಲಗೆ ಬದಲಾಯಿಸು.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "ಬહુಮತಿ",
          "desc":
              "ಎವರಿಗಾದರೂ ಒಂದು ಚಿಕ್ಕ ಗಿಫ್ಟ್ ಅಥವಾ ತಿನ್ನುವ ವಸ್ತು ಕೊಟ್ಟು 'ನೀನು ಇದು ಇಷ್ಟಪಡುತ್ತೀಯಾ ಎಂದು ಕೊಂಡಿದ್ದೆ' ಎಂದು ಹೇಳು.",
        },
        {
          "id": "B2",
          "title": "ಧೈರ್ಯವಂತಾದ ನಾಯಕತ್ವ",
          "desc":
              "ಒಂದು ಚಿಕ್ಕ ಗ್ರೂಪ್ ವ್ಯಕ್ತಿಗಳಿಗೆ ಒಂದು ಪ್ಲಾನ್ ಅಥವಾ ಯಾವುದಾದರೂ ಸ್ಥಳವನ್ನು ಸಂದರ್ಶಿಸುವ ಆಲೋಚನೆಯನ್ನು ಪ್ರತಿಪಾದಿಸು.",
        },
        {
          "id": "B3",
          "title": "ವಿಲ್ಯ ವ್ಯಕ್ತಪಡಿಸುವುದು",
          "desc":
              "ನೀ ಜೀವನದಲ್ಲಿ ಅವರ ಉనికిని ನೀನು ಏಕೆ ಮೌಲ್ಯಯುತವಾದದ್ದು ಎಂದು ಭಾವಿಸುತ್ತಿದ್ದೀಯಾ ಎಂದು ಒಬ್ಬರಿಗೆ ಪ್ರತ್ಯೇಕವಾಗಿ ಹೇಳು.",
        },
        {
          "id": "B4",
          "title": "ಸಾಮಾಜಿಕ ನಿರ್ವಾಹಕ",
          "desc":
              "ಕೊಂತಮಂದಿ ವ್ಯಕ್ತಿಗಳಿಗಾಗಿ ಒಂದು ಚಿಕ್ಕ ಮೀಟಿಂಗ್ ಅಥವಾ ಒಂದು ಕಾಫಿ ಡೇಟ್ ಅನ್ನು ಏರ್ಪಾಟು ಮಾಡು.",
        },
        {
          "id": "B5",
          "title": "ಲೋತೈನ ಸಂಭಾಷಣೆ",
          "desc":
              "ಒಬ್ಬರೊಂದಿಗೆ 15 ನಿಮಿಷಗಳಿಗಿಂತ ಹೆಚ್ಚು ಸಮಯದ ಪಾಲು ಲೋತೈನ ಮತ್ತು ಅರ್ಥವಂತಾದ ಸಂಭಾಷಣೆಯನ್ನು ಸಾಗಿಸು.",
        },
        {
          "id": "B6",
          "title": "ಆತ್ಮವಿಶ್ವಾಸ ಶಿಖರ",
          "desc":
              "ನಿನ್ನನ್ನು ನೋಡಿದಾಗ ಕಾಸ್ತ ಭಯ ಅಥವಾ ವೆನಕಡುಗು ಹಾಕುವಂತೆ ಮಾಡುವ ಒಬ್ಬರೊಂದಿಗೆ ಸಂಭಾಷಣೆಯನ್ನು ಪ್ರಾರಂಭಿಸು.",
        },
        {
          "id": "B7",
          "title": "ಬಹಿರಂಗ ಶುಭಾಕಾಂಕ್ಷೆ",
          "desc":
              "ಒಂದು ಗ್ರೂಪ್‌ನಲ್ಲಿ ಒಂದು ನಿರ್ದಿಷ್ಟ ವ್ಯಕ್ತಿಗಾಗಿ ಒಂದು ಚಿಕ್ಕ, ಸಕಾರಾತ್ಮಕ ಪ್ರಶಂಸೆ ವಾಕ್ಯವನ್ನು ಹೇಳು ಅಥವಾ ಅವರ ಪ್ರಯತ್ನವನ್ನು ಸ್ವಾಗತಿಸು.",
        },
        {
          "id": "B8",
          "title": "ಸರಿಹದ್ದುಗಳು ಗೀಳುವವನು",
          "desc":
              "ಎಂತಹ ಸುದೀರ್ಘ ವಿವರಣೆ ಇಲ್ಲದೆ, ಒಂದು ಅಭ್ಯರ್ಥನೆಗೆ ಗಟ್ಟಿಯಾಗಿ ಆದರೆ ಕನಿಷ್ಠ ಕರುಣೆಯೊಂದಿಗೆ 'ಇಲ್ಲ' ಎಂದು ಹೇಳು.",
        },
        {
          "id": "B9",
          "title": "ನೇರವಾಗಿ ಅಭ್ಯರ್ಥನೆ",
          "desc":
              "ನೀನು ಇಷ್ಟಪಡುವ ಅಥವಾ ಗೌರವಿಸುವ ಒಬ್ಬರೊಂದಿಗೆ 10 ನಿಮಿಷಗಳ ಮಾತುಗಳು ಅಥವಾ ಮಾರ್ಗದರ್ಶನಕ್ಕಾಗಿ ಅಭ್ಯರ್ಥನೆ ಇಡು.",
        },
        {
          "id": "B10",
          "title": "ಭಾವೋದ್ವೇಗ ಪ್ರಯತ್ನ",
          "desc":
              "ನೀ ಸ್ನೇಹಿತನು ಒಬ್ಬರೊಂದಿಗೆ ಭಾವನೆಗಳು ಅಥವಾ ಮಾನಸಿಕ ಆರೋಗ್ಯದ ಕುರಿತು ಲೋತೈನ ಚರ್ಚೆಯನ್ನು ಪ್ರಾರಂಭಿಸು.",
        },
        {
          "id": "B11",
          "title": "ಸಾಮಾಜಿಕ ಮಧ್ಯವರ್ತಿ",
          "desc":
              "ಪ್ರಶಾಂತವಾದ ಸಂಭಾಷಣೆ ದ್ವಾರ ಇಬ್ಬರು ವ್ಯಕ್ತಿಗಳ ಮಧ್ಯೆ ಒಂದು ಚಿಕ್ಕ ವಿವಾದವನ್ನು ಪರಿಹರಿಸಲು ಸಹಾಯ ಮಾಡು.",
        },
        {
          "id": "B12",
          "title": "ಧೈರ್ಯವಂತಾದ ಪ್ರಶಂಸೆ",
          "desc":
              "ಪೂರ್ತಿಯಾಗಿ ಅಪರಿಚಿತ ವ್ಯಕ್ತಿಯಾದ ಒಬ್ಬರೊಂದಿಗೆ ನೀನು ನಿಜವಾಗಿಯೂ ಅವರಲ್ಲಿ ಪ್ರಶಂಸಿಸುವ ಒಂದು ವಿಷಯವನ್ನು ಹೇಳು.",
        },
        {
          "id": "B13",
          "title": "ನೆಟ್‌ವರ್ಕಿಂಗ್ ಅಡಿ",
          "desc":
              "ನೀ ರಂಗಕ್ಕೆ ಸೇರಿದ ಒಂದು ನಿಪುಣನು ಅಥವಾ ಅರ್ಹತೆ ಗಲ ವ್ಯಕ್ತಿಯೊಂದಿಗೆ ನಿನ್ನನ್ನು ನೀನು ಪರಿಚಯ ಮಾಡಿಕೊಂಡು ಸಲಹೆ ಕೇಳು.",
        },
        {
          "id": "B14",
          "title": "ಧೈರ್ಯವಂತಾದ ನಿಜ",
          "desc":
              "ಒಬ್ಬರೊಂದಿಗೆ ಹೇಳಲು ಕಷ್ಟವಾದ ಆದರೆ ಬಂಧಕ್ಕೆ ಉಪಯೋಗ ಪಡುವ ಒಂದು ನಿಜ್ಜವನ್ನು ಹೇಳು.",
        },
        {
          "id": "B15",
          "title": "ಪೂರ್ಣ ವಿಕಾಸ",
          "desc":
              "ಒಂದು ಚಿಕ್ಕ ಸಾಮಾಜಿಕ ಕಾರ್ಯಕ್ರಮವನ್ನು ಏರ್ಪಾಟು ಮಾಡಿ, ಅಕ್ಕಡಿಕಿ ಬರುವ ಪ್ರತಿ ಅತಿಥಿ ಸೌಕರ್ಯವಾಗಿ ಇರುವಂತೆ ನೋಡು.",
        },
        {
          "id": "B16",
          "title": "ಬಹಿರಂಗ ಸ್ಪೀಕರ್",
          "desc":
              "ಒಂದು ಮೀಟಿಂಗ್ ಅಥವಾ ಇವೆಂಟ್ ಯೊಕ್ಕ ಒಂದು ಚಿಕ್ಕ ಭಾಗವನ್ನು ಲೀಡ್ ಮಾಡಲು ಅಥವಾ ಮಾತನಾಡಲು ನೀನುಗ ಮುಂದು ಬಾ.",
        },
        {
          "id": "B17",
          "title": "ಅನುಭವಗಳ ಗೈಡ್",
          "desc":
              "ಮರೊಕರಿನಿ ಪ್ರೋತ್ಸಾಹಿಸಲು ನೀ ಕಷ್ಟವಾದ ಪರಿಸ್ಥಿತಿ ಅಥವಾ ನೀನು ದಾಟಿನ ಓಟಮಿ ಅನುಭవాన్ని ಪంచుಕೊ.",
        },
        {
          "id": "B18",
          "title": "ಧೈರ್ಯವಂತಾದ ಕ್ಷಮಾಪಣೆ",
          "desc":
              "ಗತದಲ್ಲಿನ ಒಂದು ತಪ್ಪು ಕಾಗಿ ಕ್ಷಮಾಪಣೆ ಕೇಳಲು ನೀನುಗ ಸಂಭಾಷಣೆಯನ್ನು ಪ್ರಾರಂಭಿಸು, ಅದು ಎಂತ ಪಾತದೈನ ಸರೆ.",
        },
        {
          "id": "B19",
          "title": "ಮೆಂಟರ್",
          "desc":
              "ನೀಕಂತೆ ಅನುಭವದಲ್ಲಿ ತಕ್ಕುವ ಇರುವ ಒಂದು ವ್ಯಕ್ತಿಗೆ ಒಂದು ಸ್ಕಿಲ್ ಅಥವಾ ಪನಿಯಲ್ಲಿ ಸಹಾಯ ಮಾಡಲು ಮುಂದುವರಿ.",
        },
        {
          "id": "B20",
          "title": "ಸಾಮಾಜಿಕ ಆರ್ಕಿಟೆಕ್ಟ್",
          "desc":
              "ಸ್ನೇಹಿತರ ಗ್ರೂಪ್ ಕೋಸಂ ಒಂದು ಹೊಸ ಸಮಾಜಿಕ ಸಾಂಪ್ರದಾಯವನ್ನು ಅಥವಾ ತರಚು ಜರಿಗೇ ಒಂದು ಮೀಟಪ್ ಅನ್ನು ಪ್ರಾರಂಭಿಸು.",
        },
      ],
    },
    'ml': {
      "Seedling": [
        {
          "id": "S1",
          "title": "ആദ്യത്തെ ചുവട്",
          "desc":
              "ഇന്ന് ഒരാളുമായി കണ്ണ് പരസ്പരം ഉടക്കി നോക്കുകയും പുഞ്ചിരിക്കുകയും ചെയ്യുക.",
        },
        {
          "id": "S2",
          "title": "ഒരു ലളിതമായ ഹലോ",
          "desc": "അയൽക്കാരനോട് 'സുപ്രഭാതം' അല്ലെങ്കിൽ 'ഹലോ' എന്ന് പറയുക.",
        },
        {
          "id": "S3",
          "title": "നന്ദി പ്രകാശനം",
          "desc": "ഒരു കടയുടമയോട് വ്യക്തമായി 'നന്ദി' എന്ന് പറയുക.",
        },
        {
          "id": "S4",
          "title": "നിരീക്ഷണം",
          "desc":
              "ഒരു അപരിചിതനിൽ എന്തെങ്കിലും നല്ല കാര്യം ശ്രദ്ധിക്കുകയും പുഞ്ചിരിക്കുകയും ചെയ്യുക.",
        },
        {
          "id": "S5",
          "title": "നിശബ്ദമായ കൈവീശൽ",
          "desc": "ദൂരെ നിന്ന് നിങ്ങൾ തിരിച്ചറിയുന്ന ഒരാൾക്ക് നേരെ കൈവീശുക.",
        },
        {
          "id": "S6",
          "title": "വാതിൽ പിടിച്ചു കൊടുക്കൽ",
          "desc":
              "നിങ്ങളുടെ പിന്നാലെ വരുന്ന ഒരാൾക്കായി വാതിൽ തുറന്ന് പിടിക്കുക.",
        },
        {
          "id": "S7",
          "title": "തലയാട്ടൽ",
          "desc":
              "കടന്നുപോകുമ്പോൾ ഒരു സഹപ്രവർത്തകന് നേരെ സൗഹൃദത്തോടെ തലയാട്ടി അഭിവാദ്യം ചെയ്യുക.",
        },
        {
          "id": "S8",
          "title": "കണ്ണാടി പരിശീലനം",
          "desc":
              "കണ്ണാടിക്ക് മുന്നിൽ നിന്ന് 1 മിനിറ്റ് നേരം നിങ്ങളുടെ 'ആത്മവിശ്വാസമുള്ള പുഞ്ചിരി' പരിശീലിക്കുക.",
        },
        {
          "id": "S9",
          "title": "ഹ്രസ്വമായ നോട്ടം",
          "desc":
              "ഒരാളെ 2 സെക്കൻഡ് നേരം നോക്കുക, പുഞ്ചിരിച്ച ശേഷം നോട്ടം മാറ്റുക.",
        },
        {
          "id": "S10",
          "title": "നിശബ്ദമായ പ്രശംസ",
          "desc":
              "ആരുടെയെങ്കിലും സോഷ്യൽ മീഡിയ പോസ്റ്റിന് താഴെ നല്ലൊരു കമന്റ് എഴുതുക.",
        },
        {
          "id": "S11",
          "title": "ഇടം പങ്കിടൽ",
          "desc":
              "ഒരു പൊതു സ്ഥലത്ത് ഒരാളുടെ അടുത്ത് ഇരിക്കുക, ഉടൻ തന്നെ നോട്ടം മാറ്റാതെ കുറച്ചു സമയം ചെലവഴിക്കുക.",
        },
        {
          "id": "S12",
          "title": "ലളിതമായ അനുവാദം",
          "desc":
              "വഴിയിൽ വെച്ച് ഒരാളെ കടന്നുപോകുമ്പോൾ മര്യാദയോടെ 'എക്സ്ക്യൂസ് മീ' എന്ന് പറയുക.",
        },
        {
          "id": "S13",
          "title": "ഊഷ്മളമായ അഭിവാദ്യം",
          "desc": "ഒരു ഡെലിവറി ബോയിയോടോ കൊറിയർ വ്യക്തിയോടോ 'ഹലോ' എന്ന് പറയുക.",
        },
        {
          "id": "S14",
          "title": "ചെറിയൊരു സന്ദേശം",
          "desc":
              "ഒരു കുട്ടിക്കോ അല്ലെങ്കിൽ (ഉടമസ്ഥന്റെ അനുവാദത്തോടെ) ഒരു വളർത്തുമൃഗത്തിനോ നേരെ കൈവീശുക.",
        },
        {
          "id": "S15",
          "title": "മൃദുവായ പുഞ്ചിരി",
          "desc": "ഇന്ന് മൂന്ന് വ്യത്യസ്ത വ്യക്തികളെ നോക്കി പുഞ്ചിരിക്കുക.",
        },
        {
          "id": "S16",
          "title": "ഐ കോൺടാക്റ്റ് ചലഞ്ച്",
          "desc":
              "ക്യാഷ്യർ ആദ്യം നോട്ടം മാറ്റുന്നത് വരെ അവരുമായി കണ്ണ് പരസ്പരം ഉടക്കി നോക്കുക.",
        },
        {
          "id": "S17",
          "title": "ശാന്തമായ ശ്വാസോച്ഛ്വാസം",
          "desc":
              "ഇന്ന് ഏതെങ്കിലും സാമൂഹിക ഇടത്തിലേക്ക് കടക്കുന്നതിന് മുൻപ് 3 തവണ ദീർഘമായി ശ്വാസമെടുക്കുക.",
        },
        {
          "id": "S18",
          "title": "സാന്നിധ്യം",
          "desc":
              "ആളുകൾ കൂടിക്കിടക്കുന്ന ഒരിടത്ത് ഫോൺ ഒട്ടും നോക്കാതെ 5 മിനിറ്റ് നേരം നിൽക്കുക.",
        },
        {
          "id": "S19",
          "title": "അപ്രതീക്ഷിത തലയാട്ടൽ",
          "desc": "നിങ്ങളെ നോക്കുന്ന ഒരു അപരിചിതനെ നോക്കി തലയാട്ടി പകരുക.",
        },
        {
          "id": "S20",
          "title": "നല്ലൊരു യാത്രയയപ്പ്",
          "desc":
              "കടയിൽ നിന്ന് ഇറങ്ങുമ്പോൾ ആരോടെങ്കിലും 'നല്ലൊരു ദിവസം ആശംസിക്കുന്നു' എന്ന് പറയുക.",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "പ്രശംസിക്കൽ",
          "desc": "സഹപ്രവർത്തകനെയോ സഹപാഠിയെയോ മനസ്സ് തുറന്ന് പ്രശംസിക്കുക.",
        },
        {
          "id": "SP2",
          "title": "ചോദ്യം ചോദിക്കൽ",
          "desc": "ഒരു അപരിചിതനോട് സമയമോ വഴിയോ ചോദിക്കുക.",
        },
        {
          "id": "SP3",
          "title": "ചെറിയൊരു സംഭാഷണം",
          "desc":
              "ഒരാളോട് 'ഇന്ന് ദിവസം എങ്ങനെ പോകുന്നു?' എന്ന് ചോദിച്ച്, അവരുടെ മറുപടി ശ്രദ്ധയോടെ കേൾക്കുക.",
        },
        {
          "id": "SP4",
          "title": "സഹായം തേടൽ",
          "desc":
              "ഒരു പ്രത്യേക സാധനം കണ്ടെത്തുന്നതിന് സഹായിക്കാൻ സ്റ്റോർ ജീവനക്കാരനോട് അഭ്യർത്ഥിക്കുക.",
        },
        {
          "id": "SP5",
          "title": "ഓർഡർ നൽകുമ്പോൾ",
          "desc":
              "ഭക്ഷണമോ പാനീയമോ ഓർഡർ ചെയ്ത്, അവിടെയുള്ള ജീവനക്കാരോട് സുഖവിവരം അന്വേഷിക്കുക.",
        },
        {
          "id": "SP6",
          "title": "പരിചയപ്പെടൽ",
          "desc":
              "നിങ്ങളുടെ പരിസരത്തുള്ള ഒരു പുതിയ വ്യക്തിയോട് സ്വയം പരിചയപ്പെടുത്തുക.",
        },
        {
          "id": "SP7",
          "title": "കാലാവസ്ഥാ സംസാരം",
          "desc":
              "വരിയിൽ കാത്തുനിൽക്കുമ്പോൾ അടുത്തുള്ള ആളോട് കാലാവസ്ഥയെക്കുറിച്ച് സംസാരിക്കുക.",
        },
        {
          "id": "SP8",
          "title": "സാധാരണ അന്വേഷണം",
          "desc":
              "സഹപ്രവർത്തകനോട് 'വാരാന്ത്യത്തിൽ എന്താണ് ചെയ്തത്?' എന്ന് ചോദിക്കുക.",
        },
        {
          "id": "SP9",
          "title": "സഹായം വാഗ്ദാനം ചെയ്യൽ",
          "desc":
              "ആരെങ്കിലും ബുദ്ധിമുട്ടുന്നത് കണ്ടാൽ, 'ഞാൻ സഹായിക്കട്ടെ?' എന്ന് ചോദിക്കുക.",
        },
        {
          "id": "SP10",
          "title": "അഭിപ്രായം",
          "desc":
              "ഒരു ചെറിയ വസ്തുകാണിച്ച് സുഹൃത്തിനോട് 'ഇതിനെക്കുറിച്ച് നീ എന്ത് കരുതുന്നു?' എന്ന് ചോദിക്കുക.",
        },
        {
          "id": "SP11",
          "title": "സ്ഥിരീകരണം",
          "desc":
              "ഒരു അപരിചിതനുമായി ഒരു കാര്യം സ്ഥിരീകരിക്കുക (ഉദാ: 'ഇതാണോ ശരിയായ വരി?').",
        },
        {
          "id": "SP12",
          "title": "പരിസരത്തെക്കുറിച്ച്",
          "desc":
              "ചുറ്റുമുള്ള അന്തരീക്ഷത്തെക്കുറിച്ച് ചെറിയൊരു അഭിപ്രായം പറയുക (ഉദാ: 'ഇവിടെ നല്ല തിരക്കാണല്ലോ').",
        },
        {
          "id": "SP13",
          "title": "ചെറിയൊരു സഹായം",
          "desc":
              "മേശപ്പുറത്തിരിക്കുന്ന ഒരു സാധനം (നാപ്കിൻ പോലുള്ളവ) നിങ്ങളുടെ നേരെ നീക്കിത്തരാൻ ആരോടെങ്കിലും ആവശ്യപ്പെടുക.",
        },
        {
          "id": "SP14",
          "title": "നല്ല ഫീഡ്‌ബാക്ക്",
          "desc":
              "ഭക്ഷണം വളരെ മികച്ചതായിരുന്നു എന്ന് ഇറങ്ങുന്നതിന് മുൻപ് വെയിറ്ററോട് പറയുക.",
        },
        {
          "id": "SP15",
          "title": "സാധാരണ പലകരിക്കൽ",
          "desc":
              "ഒരു മാസമായി സംസാരിക്കാത്ത വ്യക്തിക്ക് 'എങ്ങനെയുണ്ട്, സുഖമാണോ?' എന്ന് മെസ്സേജ് അയക്കുക.",
        },
        {
          "id": "SP16",
          "title": "തുറന്ന ചോദ്യം",
          "desc":
              "ആരോടെങ്കിലും 'ഈ നഗരത്തിൽ നിങ്ങൾക്ക് ഏറ്റവും ഇഷ്ടപ്പെട്ട സ്ഥലം ഏതാണ്?' എന്ന് ചോദിക്കുക.",
        },
        {
          "id": "SP17",
          "title": "ഏറ്റവും ചെറിയ റിസ്ക്",
          "desc":
              "അടുത്ത് വാഷ്‌റൂം എവിടെയുണ്ടെന്ന് അറിയാമോ എന്ന് ഒരു അപരിചിതനോട് ചോദിക്കുക.",
        },
        {
          "id": "SP18",
          "title": "വസ്തുക്കളുടെ പ്രശംസ",
          "desc": "ആരോടെങ്കിലും അവരുടെ ഷൂസ്/ബാഗ്/ആക്സസറി കൊള്ളാമെന്ന് പറയുക.",
        },
        {
          "id": "SP19",
          "title": "മര്യാദപൂർവ്വമായ കാത്തിരിപ്പ്",
          "desc":
              "ആരോടെങ്കിലും മറുപടി നൽകുന്നതിന് മുൻപ് അവരുടെ സംസാരം പൂർണ്ണമായി തീരുന്നത് വരെ കാത്തിരിക്കുക.",
        },
        {
          "id": "SP20",
          "title": "സൗഹൃദപരമായ വിടപറച്ചിൽ",
          "desc":
              "ഇപ്പോൾ നിങ്ങളോട് ചെറിയ രീതിയിൽ സംസാരിച്ച വ്യക്തിക്ക് നേരെ കൈവീശി 'ബൈ' പറയുക.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "അഭിപ്രായങ്ങൾ അന്വേഷിക്കുന്നയാൾ",
          "desc":
              "ഒരു പുസ്തകം, സിനിമ അല്ലെങ്കിൽ പാട്ട് എന്നിവയെക്കുറിച്ചുള്ള അവരുടെ അഭിപ്രായം ചോദിക്കുക.",
        },
        {
          "id": "L2",
          "title": "വിവരങ്ങൾ",
          "desc":
              "ഒരാൾ തങ്ങളെക്കുറിച്ച് എന്തെങ്കിലും പറഞ്ഞതിന് ശേഷം അതുമായി ബന്ധപ്പെട്ട മറ്റൊരു ചോദ്യം കൂടി ചോദിക്കുക.",
        },
        {
          "id": "L3",
          "title": "ശിപാർശ",
          "desc":
              "അടുത്ത് ഭക്ഷണം കഴിക്കാൻ നല്ല ഹോട്ടൽ ഏതെങ്കിലും ഉണ്ടോ എന്ന് ഒരു അപരിചിതനോട് ചോദിക്കുക.",
        },
        {
          "id": "L4",
          "title": "പൊതുവായ കണക്ട്",
          "desc":
              "ഒരാളുമായി ഒരു പൊതുവായ താല്പര്യം കണ്ടെത്തുകയും അതിനെക്കുറിച്ച് 2 മിനിറ്റ് സംസാരിക്കുകയും ചെയ്യുക.",
        },
        {
          "id": "L5",
          "title": "സഹായ ഹസ്തം",
          "desc":
              "ഒരു ചെറിയ കാര്യത്തിൽ (ബാഗ് ചുമക്കുന്നത് പോലെ) സഹായിക്കാൻ മുന്നോട്ട് വരിക.",
        },
        {
          "id": "L6",
          "title": "സാമൂഹിക നിരീക്ഷണം",
          "desc":
              "നിങ്ങൾ രണ്ടുപേരുടെയും ചുറ്റും നടക്കുന്ന ഒരു സംഭവത്തെ അടിസ്ഥാനമാക്കി സംഭാഷണം ആരംഭിക്കുക.",
        },
        {
          "id": "L7",
          "title": "തുറന്ന ചോദ്യം",
          "desc":
              "ആരോടെങ്കിലും 'നിങ്ങൾ ഈ തൊഴിലിലേക്ക് അല്ലെങ്കിൽ ജോലിയിലേക്ക് എങ്ങനെയാണ് വന്നത്?' എന്ന് ചോദിക്കുക.",
        },
        {
          "id": "L8",
          "title": "സജീവമായി കേൾക്കുന്നയാൾ",
          "desc":
              "ഒരാളുടെ സംസാരം തടസ്സപ്പെടുത്താതെ 3 മിനിറ്റ് കേൾക്കുക, അതിനുശേഷം അവർ പറഞ്ഞത് ചുരുക്കി പറയുക.",
        },
        {
          "id": "L9",
          "title": "പൊതുവായ ചിരി",
          "desc":
              "ഒരു ചെറിയ സംഘത്തിന് ഒരു ചെറിയ, ഹാസ്യഭരിതമായ കഥയോ ജോക്കോ പറഞ്ഞു കൊടുക്കുക.",
        },
        {
          "id": "L10",
          "title": "കൗതുകം",
          "desc":
              "ആരോടെങ്കിലും അവർ ഏത് നാട്ടുകാരനാണെന്നും ആ നാട്ടിൽ അവർക്ക് എന്താണ് ഇഷ്ടമെന്നും ചോദിക്കുക.",
        },
        {
          "id": "L11",
          "title": "യഥാർത്ഥ താല്പര്യം",
          "desc":
              "സഹപ്രവർത്തകനോട് ജോലിക്ക് വെളിയിലുള്ള അവരുടെ ഹോബികളെക്കുറിച്ച് ചോദിക്കുക.",
        },
        {
          "id": "L12",
          "title": "മൃദുവായ ഉപദേശം",
          "desc":
              "നിങ്ങൾ വിദഗ്ദ്ധനായ ഒരു കാര്യത്തെക്കുറിച്ച് ഒരാൾക്ക് ഉപയോഗപ്രദമായ ഉപദേശം നൽകുക.",
        },
        {
          "id": "L13",
          "title": "ഗ്രൂപ്പിൽ പിന്തുണ",
          "desc":
              "ഒരു ചെറിയ ഗ്രൂപ്പ് ചർച്ചയിൽ ഒരാളുടെ അഭിപ്രായത്തെ പിന്തുണച്ച് തലയാട്ടുക.",
        },
        {
          "id": "L14",
          "title": "ലളിതമായ ക്ഷണം",
          "desc":
              "ആരോടെങ്കിലും 'ഞങ്ങളോടൊപ്പം ഉച്ചഭക്ഷണത്തിന് കൂടുന്നോ?' എന്ന് ചോദിക്കുക.",
        },
        {
          "id": "L15",
          "title": "ആത്മാർത്ഥമായ ചിന്ത",
          "desc":
              "ഒരാളോട് 'നിങ്ങൾ X ചെയ്തപ്പോൾ എനിക്ക് വളരെ സന്തോഷം തോന്നി' എന്ന് പറഞ്ഞ് അത് എന്തുകൊണ്ടെന്ന് വ്യക്തമാക്കുക.",
        },
        {
          "id": "L16",
          "title": "കൗതുകത്തിന്റെ ഇടവേള",
          "desc":
              "ആരോടെങ്കിലും 'ഞാൻ എപ്പോഴും ആലോചിക്കാറുണ്ട്, X യഥാർത്ഥത്തിൽ എങ്ങനെയാണ് പ്രവർത്തിക്കുന്നത്?' എന്ന് ചോദിക്കുക.",
        },
        {
          "id": "L17",
          "title": "ചെറിയ ഗ്രൂപ്പ് ലീഡ്",
          "desc":
              "ഒരു ഗ്രൂപ്പിൽ രണ്ടോ മൂന്നോ പേർ ഉത്തരം പറയേണ്ട ഒരു ചോദ്യം ചോദിക്കുക.",
        },
        {
          "id": "L18",
          "title": "യഥാർത്ഥ അഭിനന്ദനം",
          "desc":
              "ഒരാളുടെ വ്യക്തിത്വ സവിശേഷതയെ അഭിനന്ദിക്കുക (ഉദാഹരണത്തിന്, 'നിങ്ങൾ ഒരു മികച്ച ശ്രോതാവാണ്').",
        },
        {
          "id": "L19",
          "title": "പങ്കുവെച്ച അനുഭവം",
          "desc":
              "സംഭാഷണത്തിനിടയിൽ 'ഞാനും ആ സാഹചര്യത്തിലൂടെ കടന്നുപോയിട്ടുണ്ട്' എന്ന് പറയുക.",
        },
        {
          "id": "L20",
          "title": "അർത്ഥവത്തായ നിശബ്ദത",
          "desc":
              "സംഭാഷണത്തിൽ പെട്ടെന്ന് നിശബ്ദത ഉണ്ടായാൽ അത് വാക്കുകൾ കൊണ്ട് വേഗം നികത്താൻ ശ്രമിക്കാതെ സ്വാഭാവികമായി വിടുക.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "ധീരമായ തുടക്കം",
          "desc":
              "നിങ്ങൾക്ക് അത്ര നന്നായി അറിയാത്ത ഒരാളോട് സംഭാഷണം ആരംഭിക്കുക.",
        },
        {
          "id": "ST2",
          "title": "ആത്മാർത്ഥമായ പങ്കുവെക്കൽ",
          "desc":
              "ഒരു ഗ്രൂപ്പിൽ നിങ്ങളുടെ ഒരു ചെറിയ വ്യക്തിഗത കഥയോ അഭിപ്രായമോ പങ്കുവെക്കുക.",
        },
        {
          "id": "ST3",
          "title": "അഭിപ്രായവ്യത്യാസം",
          "desc":
              "ആരുടെയെങ്കിലും അഭിപ്രായത്തോട് മാന്യമായി വിയോജിക്കുകയും അത് എന്തുകൊണ്ടെന്ന് വിശദീകരിക്കുകയും ചെയ്യുക.",
        },
        {
          "id": "ST4",
          "title": "ഗ്രൂപ്പിൽ ചേരൽ",
          "desc":
              "നടന്നുകൊണ്ടിരിക്കുന്ന ഒരു ഗ്രൂപ്പ് സംഭാഷണത്തിൽ പങ്കുചേരുകയും ചിന്തനീയമായ ഒരു വാചകം കൂട്ടിച്ചേർക്കുകയും ചെയ്യുക.",
        },
        {
          "id": "ST5",
          "title": "ചർച്ചാ വിഷയം നൽകൽ",
          "desc":
              "ഒരു സോഷ്യൽ ഗ്രൂപ്പിൽ ചർച്ചയ്ക്കായി ഒരു പുതിയ വിഷയം അവതരിപ്പിക്കുക.",
        },
        {
          "id": "ST6",
          "title": "പൊതുവേദിയിലെ ചോദ്യം",
          "desc":
              "ഒരു പൊതുയോഗത്തിലോ ക്ലാസ് റൂം പരിസ്ഥിതിയിലോ ഒരു ചോദ്യം ചോദിക്കുക.",
        },
        {
          "id": "ST7",
          "title": "ധീരമായ അഭ്യർത്ഥന",
          "desc":
              "ഒരു കഫേയിലോ പാർക്കിലോ വെച്ച് ഒരു അപരിചിതനോട് അവരുടെ അടുത്ത് ഇരിക്കാമോ എന്ന് ചോദിക്കുക.",
        },
        {
          "id": "ST8",
          "title": "സംഭാഷണത്തിന്റെ പാലം",
          "desc":
              "പരസ്പരം അറിയാത്ത രണ്ട് ആളുകളെ പരിചയപ്പെടുത്തുകയും അവർ തമ്മിൽ ഒരു പൊതുവായ കാര്യം കണ്ടെത്തുകയും ചെയ്യുക.",
        },
        {
          "id": "ST9",
          "title": "ആത്മവിശ്വാസത്തോടെയുള്ള ആവശ്യം",
          "desc":
              "നിങ്ങളെ ബുദ്ധിമുട്ടിക്കുന്ന എന്തെങ്കിലും ചെയ്യുന്നത് നിർത്താനോ മാറിക്കൊടുക്കാനോ ഒരാളോട് മാന്യമായി ആവശ്യപ്പെടുക.",
        },
        {
          "id": "ST10",
          "title": "കഥ പറയുന്നയാൾ",
          "desc":
              "മൂന്നോ അതിലധികമോ ആളുകളുള്ള ഒരു ഗ്രൂപ്പിന് മുന്നിൽ ഒരു കഥ പറയുന്നതിൽ നേതൃത്വം നൽകുക.",
        },
        {
          "id": "ST11",
          "title": "തുറന്ന വെല്ലുവിളി",
          "desc":
              "ഒരു ഗ്രൂപ്പിലെ പൊതുവായ ഒരു അഭിപ്രായത്തെ സൗഹാർദ്ദപരവും ആദരവുള്ളതുമായ രീതിയിൽ ചോദ്യം ചെയ്യുക.",
        },
        {
          "id": "ST12",
          "title": "സാമൂഹിക മുൻകൈ",
          "desc":
              "ഒരു മുറിയിലേക്ക് പ്രവേശിക്കുമ്പോൾ എല്ലാവരോടും ആദ്യം 'ഹലോ' പറയുന്ന വ്യക്തിയാകുക.",
        },
        {
          "id": "ST13",
          "title": "സഹാനുഭൂതിയോടെയുള്ള കേൾവി",
          "desc":
              "ആരെങ്കിലും അവരുടെ വിഷമങ്ങളോ ദേഷ്യമോ പങ്കുവെക്കുമ്പോൾ അത് ശ്രദ്ധയോടെ കേൾക്കുകയും പിന്തുണ നൽകുന്ന മറുപടി നൽകുകയും ചെയ്യുക.",
        },
        {
          "id": "ST14",
          "title": "പൊതു അവതരണം",
          "desc":
              "ഒരു സാമൂഹിക ഒത്തുചേരലിൽ നിങ്ങൾക്ക് പ്രിയപ്പെട്ട ഒരു വിഷയത്തെക്കുറിച്ച് 1-2 മിനിറ്റ് സംസാരിക്കുക.",
        },
        {
          "id": "ST15",
          "title": "ബലഹീനത തുറന്നുപറയൽ",
          "desc":
              "ഒരു ഗ്രൂപ്പിന് മുന്നിൽ നിങ്ങൾക്ക് ഏതെങ്കിലും കാര്യത്തിൽ പരിഭ്രമമുണ്ടായിരുന്നു എന്ന് തുറന്നുപറയുകയും അതിനെക്കുറിച്ച് ഒന്നിച്ച് ചിരിക്കുകയും ചെയ്യുക.",
        },
        {
          "id": "ST16",
          "title": "അതിരുകൾ നിശ്ചയിക്കൽ",
          "desc":
              "ഒരുപാട് വിശദീകരണങ്ങൾ നൽകാതെ, പോകാൻ ആഗ്രഹിക്കാത്ത ഒരു ക്ഷണം മാന്യമായി നിരസിക്കുക.",
        },
        {
          "id": "ST17",
          "title": "സജീവ മധ്യസ്ഥൻ",
          "desc":
              "ഒരു അഭിപ്രായവ്യത്യാസത്തിൽ രണ്ട് ആളുകളെ ഒരു മധ്യസ്ഥ തീരുമാനത്തിൽ എത്തിക്കാൻ സഹായിക്കുക.",
        },
        {
          "id": "ST18",
          "title": "പരസ്യമായ അഭിനന്ദനം",
          "desc":
              "ഒരു ഗ്രൂപ്പിൽ പരസ്യമായി ഒരാളുടെ ശ്രമത്തെയോ നേട്ടത്തെയോ അഭിനന്ദിക്കുക.",
        },
        {
          "id": "ST19",
          "title": "നേരിട്ടുള്ള സമീപനം",
          "desc":
              "നിങ്ങൾക്ക് ആവശ്യമുള്ള ഒരു സഹായത്തിനോ ഉപദേശത്തിനോ വേണ്ടി ഒരാളോട് നേരിട്ട് അഭ്യർത്ഥിക്കുക.",
        },
        {
          "id": "ST20",
          "title": "സംഭാഷണത്തിന്റെ മോഡ് മാറ്റൽ",
          "desc":
              "ഒരു സംഭാഷണത്തെ വിരസമായ ഒരു വിഷയത്തിൽ നിന്ന് രസകരമായ ഒരു വിഷയത്തിലേക്ക് സുഗമമായി മാറ്റുക.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "സമ്മാനം",
          "desc":
              "ആർക്കെങ്കിലും ഒരു ചെറിയ സമ്മാനമോ പലഹാരമോ നൽകി 'ഇത് നിങ്ങൾക്ക് ഇഷ്ടപ്പെടുമെന്ന് ഞാൻ കരുതി' എന്ന് പറയുക.",
        },
        {
          "id": "B2",
          "title": "ധീരമായ നേതൃത്വം",
          "desc":
              "ആളുകളുടെ ഒരു ചെറിയ ഗ്രൂപ്പിന് ഒരു പദ്ധതിയോ എവിടെയെങ്കിലും പോകാനുള്ള ആശയമോ മുന്നോട്ട് വെക്കുക.",
        },
        {
          "id": "B3",
          "title": "കൃതജ്ഞത പ്രകടിപ്പിക്കൽ",
          "desc":
              "ഒരാളോട് കൃത്യമായി പറയുക, നിങ്ങളുടെ ജീവിതത്തിൽ അവരുടെ സാന്നിധ്യം എത്രമാത്രം വിലപ്പെട്ടതാണെന്ന്.",
        },
        {
          "id": "B4",
          "title": "സാമൂഹിക സംഘാടകൻ",
          "desc":
              "കുറച്ചുപേർക്കായി ഒരു ചെറിയ ഒത്തുചേരലോ കോഫി ഡേറ്റോ സംഘടിപ്പിക്കുക.",
        },
        {
          "id": "B5",
          "title": "ആഴത്തിലുള്ള സംഭാഷണം",
          "desc":
              "ആരോടെങ്കിലും 15 മിനിറ്റിലധികം നീണ്ടുനിൽക്കുന്ന ആഴമേറിയതും അർത്ഥവത്തായതുമായ ഒരു സംഭാഷണം നടത്തുക.",
        },
        {
          "id": "B6",
          "title": "ആത്മവിശ്വാസത്തിന്റെ കൊടുമുടി",
          "desc":
              "നിങ്ങളെ നോക്കുമ്പോൾ അല്പം ഭയമോ മടിയോ തോന്നുന്ന ഒരാളോട് സംഭാഷണം ആരംഭിക്കുക.",
        },
        {
          "id": "B7",
          "title": "പരസ്യമായ ആശംസ",
          "desc":
              "ഒരു ഗ്രൂപ്പിൽ ഒരു നിർദ്ദിഷ്ട വ്യക്തിക്കായി ഒരു ചെറിയ, പോസിറ്റീവായ പ്രശംസാ വാചകം പറയുക അല്ലെങ്കിൽ അവരുടെ ശ്രമത്തെ സ്വാഗതം ചെയ്യുക.",
        },
        {
          "id": "B8",
          "title": "അതിരുകൾ നിശ്ചയിക്കുന്നയാൾ",
          "desc":
              "ഒരുപാട് വിശദീകരണങ്ങൾ നൽകാതെ, ഒരു ആവശ്യത്തോട് ദൃഢമായി എന്നാൽ കനിവോടെ 'ഇല്ല' എന്ന് പറയുക.",
        },
        {
          "id": "B9",
          "title": "നേരിട്ടുള്ള അഭ്യർത്ഥന",
          "desc":
              "നിങ്ങൾ ഇഷ്ടപ്പെടുന്നതോ ബഹുമാനിക്കുന്നതോ ആയ ഒരാളോട് 10 മിനിറ്റ് സംസാരിക്കാനോ മാർഗ്ഗനിർദ്ദേശത്തിനോ വേണ്ടി അഭ്യർത്ഥിക്കുക.",
        },
        {
          "id": "B10",
          "title": "വൈകാരിക മുൻകൈ",
          "desc":
              "നിങ്ങളുടെ ഒരു സുഹൃത്തുമായി വികാരങ്ങളെക്കുറിച്ചോ മാനസികാരോഗ്യത്തെക്കുറിച്ചോ ആഴത്തിലുള്ള ചർച്ചയ്ക്ക് തുടക്കമിടുക.",
        },
        {
          "id": "B11",
          "title": "സാമൂഹിക മധ്യസ്ഥൻ",
          "desc":
              "ശാന്തമായ സംഭാഷണത്തിലൂടെ രണ്ട് വ്യക്തികൾക്കിടയിലെ ഒരു ചെറിയ തർക്കം പരിഹരിക്കാൻ സഹായിക്കുക.",
        },
        {
          "id": "B12",
          "title": "ധീരമായ പ്രശംസ",
          "desc":
              "പൂർണ്ണമായും അപരിചിതനായ ഒരാളോട് നിങ്ങൾ യഥാർത്ഥത്തിൽ അവരിൽ അഭിനന്ദിക്കുന്ന ഒരു കാര്യം പറയുക.",
        },
        {
          "id": "B13",
          "title": "നെറ്റ്‌വർക്കിംഗ് ചുവട്",
          "desc":
              "നിങ്ങളുടെ മേഖലയിലെ ഒരു വിദഗ്ദ്ധനോ യോഗ്യതയുള്ള വ്യക്തിക്കോ മുന്നിൽ സ്വയം പരിചയപ്പെടുത്തുകയും ഉപദേശം ചോദിക്കുകയും ചെയ്യുക.",
        },
        {
          "id": "B14",
          "title": "ധീരമായ സത്യം",
          "desc":
              "ഒരാളോട് പറയാൻ ബുദ്ധിമുട്ടുള്ളതും എന്നാൽ ബന്ധത്തിന് ഉപകാരപ്രദവുമായ ഒരു സത്യം പറയുക.",
        },
        {
          "id": "B15",
          "title": "പൂർണ്ണ വികാസം",
          "desc":
              "ഒരു ചെറിയ സാമൂഹിക പരിപാടി സംഘടിപ്പിക്കുകയും അവിടെ വരുന്ന ഓരോ അതിഥിയും സുഖകരമായിരിക്കുന്നു എന്ന് ഉറപ്പാക്കുകയും ചെയ്യുക.",
        },
        {
          "id": "B16",
          "title": "പൊതു പ്രഭാഷകൻ",
          "desc":
              "ഒരു മീറ്റിംഗിന്റെയോ പരിപാടിയുടെയോ ഒരു ചെറിയ ഭാഗം നയിക്കാനോ സംസാരിക്കാനോ നിങ്ങളായി മുന്നോട്ട് വരിക.",
        },
        {
          "id": "B17",
          "title": "അനുഭവങ്ങളിലൂടെയുള്ള വഴികാട്ടി",
          "desc":
              "മറ്റൊരാളെ പ്രോത്സാഹിപ്പിക്കുന്നതിനായി നിങ്ങളുടെ ബുദ്ധിമുട്ടുള്ള സാഹചര്യമോ നിങ്ങൾ മറികടന്ന പരാജയത്തിന്റെ അനുഭവമോ പങ്കുവെക്കുക.",
        },
        {
          "id": "B18",
          "title": "ധീരമായ മാപ്പപേക്ഷ",
          "desc":
              "കഴിഞ്ഞകാലത്തെ ഒരു തെറ്റിന് മാപ്പ് ചോദിക്കാൻ നിങ്ങളായി സംഭാഷണം ആരംഭിക്കുക, അത് എത്ര പഴയതാണെങ്കിലും ശരി.",
        },
        {
          "id": "B19",
          "title": "മെന്റർ",
          "desc":
              "നിങ്ങളേക്കാൾ അനുഭവസമ്പത്ത് കുറഞ്ഞ ഒരു വ്യക്തിക്ക് ഒരു സ്കില്ലിലോ ജോലിയിലോ സഹായം നൽകാൻ മുന്നോട്ട് വരിക.",
        },
        {
          "id": "B20",
          "title": "സോഷ്യൽ ആർക്കിടെക്റ്റ്",
          "desc":
              "കൂട്ടുകാരുടെ ഗ്രൂപ്പിനായി എപ്പോഴും ആവർത്തിക്കാവുന്ന ഒരു പുതിയ ഒത്തുചേരൽ ശീലമോ മീറ്റപ്പോ ആരംഭിക്കുക.",
        },
      ],
    },
    'mr': {
      "Seedling": [
        {
          "id": "S1",
          "title": "पहिले पाऊल",
          "desc": "आज एका व्यक्तीशी डोळे मिळवून बघ आणि त्याच्याकडे पाहून हस.",
        },
        {
          "id": "S2",
          "title": "एक साधा नमस्कार",
          "desc": "शेजाऱ्याला 'शुभप्रभात' किंवा 'नमस्कार' असे म्हण.",
        },
        {
          "id": "S3",
          "title": "धन्यवाद",
          "desc": "दुकानदाराला स्पष्टपणे 'धन्यवाद' म्हण.",
        },
        {
          "id": "S4",
          "title": "निरीक्षण",
          "desc":
              "एका अनोळखी व्यक्तीमध्ये काहीतरी सकारात्मक शोध आणि मनापासून हस.",
        },
        {
          "id": "S5",
          "title": "शांतपणे हात हलवणे",
          "desc": "दूरवरून तू ओळखलेल्या कोणालाही हळूच हात हलवून अभिवादन कर.",
        },
        {
          "id": "S6",
          "title": "दरवाजा धरून ठेवणे",
          "desc": "तुझ्या मागे येणाऱ्या व्यक्तीसाठी दरवाजा उघडून धरून ठेव.",
        },
        {
          "id": "S7",
          "title": "फक्त मान डोलवणे",
          "desc":
              "बाजूने जाताना सहकाऱ्याला मैत्रिपूर्णपणे मान डोलवून नमस्कार कर.",
        },
        {
          "id": "S8",
          "title": "आरशासमोरील सराव",
          "desc":
              "आरशासमोर उभे राहून १ मिनिटासाठी तुझ्या 'आत्मविश्वासू हास्याचा' सराव कर.",
        },
        {
          "id": "S9",
          "title": "थोड्या वेळचा कटाक्ष",
          "desc": "कोणाकडे तरी २ सेकंद बघ, मग हसून नजर फिरवून घे.",
        },
        {
          "id": "S10",
          "title": "शांतपणे कौतुक",
          "desc": "कोणाच्या तरी सोशल मीडिया पोस्टच्या खाली एक छान कमेंट लिही.",
        },
        {
          "id": "S11",
          "title": "जागा शेअर करणे",
          "desc":
              "सार्वजनिक ठिकाणी कोणाच्या तरी शेजारी बस आणि लगेच नजर न फिरवता काही वेळ तिथे थांब.",
        },
        {
          "id": "S12",
          "title": "साधी मने राखणे",
          "desc":
              "कॉरिडॉरमध्ये कोणाला तरी ओलांडून जाताना मरण्यपूर्वक 'एक्सक्यूज मी' म्हण.",
        },
        {
          "id": "S13",
          "title": "आत्मीयतेचे स्वागत",
          "desc":
              "डिलीव्हरी बॉय किंवा कुरिअर आणणाऱ्या व्यक्तीला 'नमस्कार' म्हण.",
        },
        {
          "id": "S14",
          "title": "लहानशी सायकल",
          "desc":
              "एका लहान मुलाला किंवा (मालकाच्या परवानगीने) पाळीव प्राण्याला हात हलवून दाखव.",
        },
        {
          "id": "S15",
          "title": "मऊ हसू",
          "desc": "आज तीन वेगवेगळ्या लोकांकडे पाहून हस.",
        },
        {
          "id": "S16",
          "title": "आय कॉन्टॅक्टचे आव्हान",
          "desc": "कॅशियरने आधी नजर फिरवेपर्यंत त्याच्याशी डोळे मिळवून ठेव.",
        },
        {
          "id": "S17",
          "title": "प्रशांत श्वास",
          "desc":
              "आज कोणत्याही सामाजिक ठिकाणी जाण्यापूर्वी ३ वेळा दीर्घ श्वास घे.",
        },
        {
          "id": "S18",
          "title": "उपस्थिती",
          "desc":
              "गर्दीच्या ठिकाणी फोन अजिबात न बघता ५ मिनिटांसाठी शांतपणे उभा राहा.",
        },
        {
          "id": "S19",
          "title": "सहजपणे मान डोलवणे",
          "desc":
              "तुझ्याशी डोळे मिळवणाऱ्या अनोळखी व्यक्तीला मान डोलवून प्रतिसाद दे.",
        },
        {
          "id": "S20",
          "title": "छान निरोप",
          "desc":
              "दुकानातून बाहेर पडताना कोणाला तरी 'तुमचा दिवस चांगला जावो' असे म्हण.",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "कमलीचे कौतुक",
          "desc": "सहकाऱ्याचे किंवा क्लासमेटचे मनापासून कौतुक कर.",
        },
        {
          "id": "SP2",
          "title": "प्रश्न विचारणे",
          "desc": "एका अनोळखी व्यक्तीला वेळ किंवा रस्ता विचार.",
        },
        {
          "id": "SP3",
          "title": "छोटी चर्चा",
          "desc":
              "कोणाला तरी 'आजचा दिवस कसा चालला आहे?' असे विचार आणि त्याचे उत्तर नीट ऐकून घे.",
        },
        {
          "id": "SP4",
          "title": "मदत मागणे",
          "desc":
              "एक ठराविक वस्तू शोधण्यासाठी स्टोअरमधील कर्मचाऱ्याची मदत माग.",
        },
        {
          "id": "SP5",
          "title": "ऑर्डर देताना",
          "desc":
              "ड्रिंक किंवा फूड ऑर्डर कर आणि तिथल्या स्टाफला ते कसे आहेत हे विचार.",
        },
        {
          "id": "SP6",
          "title": "ओळख करून देणे",
          "desc":
              "तुझ्या परिसरातील एका नवीन व्यक्तीला तुझी स्वतःची ओळख करून दे.",
        },
        {
          "id": "SP7",
          "title": "हवामानाविषयी गप्पा",
          "desc":
              "लायनमध्ये वाट पाहत असताना शेजारील व्यक्तीशी हवामानाबद्दल बोल.",
        },
        {
          "id": "SP8",
          "title": "साधा शोध",
          "desc": "सहकाऱ्याला 'विकेंडला काय केलेस?' असे विचार.",
        },
        {
          "id": "SP9",
          "title": "मदत पुढे करणे",
          "desc":
              "कोणीतरी अडचणीत असल्याचे दिसल्यास, 'तुला मदतीची गरज आहे का?' असे विचार.",
        },
        {
          "id": "SP10",
          "title": "अभिप्राय घेणे",
          "desc":
              "एक छोटी वस्तू दाखवत मित्राला 'तुला याबद्दल काय वाटते?' असे विचार.",
        },
        {
          "id": "SP11",
          "title": "खात्री करणे",
          "desc":
              "एका अनोळखी व्यक्तीशी एक तपशील कन्फर्म करून घे (उदा: 'हीच लाईन बरोबर आहे ना?').",
        },
        {
          "id": "SP12",
          "title": "परिसराबद्दल चर्चा",
          "desc":
              "सभोवतालच्या वातावरणाबद्दल एक छोटी कमेंट कर (उदा: 'इथे खरंच खूप गर्दी आहे').",
        },
        {
          "id": "SP13",
          "title": "छोटीशी विनंती",
          "desc":
              "टेबलवर असताना एखादी वस्तू (जसे की नॅपकिन) तुझ्याकडे सरकवण्यास कोणालातरी सांग.",
        },
        {
          "id": "SP14",
          "title": "उत्कृष्ट फीडबॅक",
          "desc": "निघण्यापूर्वी वेटरला जेवण खूप अप्रतिम होते असे सांग.",
        },
        {
          "id": "SP15",
          "title": "सहज पंधरवड्याची विचारपूस",
          "desc":
              "एका महिन्यापासून न बोललेल्या व्यक्तीला 'कसा आहेस?' असा एक मेसेज पाठव.",
        },
        {
          "id": "SP16",
          "title": "ओपन क्वेश्चन",
          "desc":
              "कोणाला तरी 'या शहरात तुझी सर्वात आवडती फिरण्याची जागा कोणती आहे?' असे विचार.",
        },
        {
          "id": "SP17",
          "title": "अतिशय लहान रिस्क",
          "desc":
              "जवळपास वॉशरुम कुठे आहे हे ठाऊक आहे का, असे एका अनोळखी व्यक्तीला विचार.",
        },
        {
          "id": "SP18",
          "title": "वस्तूची स्तुती",
          "desc": "कोणाला तरी त्याचे शूज/बॅग/अॅक्सेसरी छान असल्याचे सांग.",
        },
        {
          "id": "SP19",
          "title": "मर्यादापूर्वक वाट पाहणे",
          "desc":
              "कोणालाही उत्तर देण्यापूर्वी त्याचे बोलणे पूर्ण संपेपर्यंत शांतपणे वाट पाहा.",
        },
        {
          "id": "SP20",
          "title": "मैत्रिपूर्ण निरोप",
          "desc":
              "आत्ताच तुझ्याशी लहानशी चर्चा केलेल्या व्यक्तीला हात हलवून 'बाय' म्हण.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "मतांचा शोधक",
          "desc": "एका पुस्तकावर, सिनेमावर किंवा गाण्यावर कोणाचे तरी मत विचार.",
        },
        {
          "id": "L2",
          "title": "तपशील विचारणे",
          "desc":
              "कोणी स्वतःबद्दल काही सांगितले असल्यास त्याला जोडून आणखी एक प्रश्न विचार.",
        },
        {
          "id": "L3",
          "title": "शिफारस",
          "desc":
              "जवळपास जेवण्यासाठी चांगले हॉटेल कोणते आहे हे सांगण्यास एका अनोळखी व्यक्तीला विचार.",
        },
        {
          "id": "L4",
          "title": "समान आवड",
          "desc": "कोणाशी तरी एक समान आवड शोधून काढ आणि त्यावर २ मिनिटे बोल.",
        },
        {
          "id": "L5",
          "title": "मदतीचा हात",
          "desc":
              "एका छोट्या कामात (बॅग उचलण्यासारख्या) मदत करण्यासाठी स्वतःहून पुढे जा.",
        },
        {
          "id": "L6",
          "title": "सामाजिक निरीक्षण",
          "desc":
              "तुम्हा दोघांच्या सभोवताली घडणाऱ्या एखाद्या घटनेच्या आधारे संभाषणाला सुरुवात कर.",
        },
        {
          "id": "L7",
          "title": "ओपन क्वेश्चन",
          "desc":
              "कोणाला तरी 'तुम्ही या व्यवसायात किंवा कामात कसे आलात?' असे विचार.",
        },
        {
          "id": "L8",
          "title": "सक्रिय ऐकणारा",
          "desc":
              "कोणाचेही बोलणे मध्येच न थांबवता ३ मिनिटे ऐकून घे, आणि नंतर त्याने काय सांगितले याचा थोडक्यात गोषवारा सांग.",
        },
        {
          "id": "L9",
          "title": "एकत्र हसणे",
          "desc": "एका लहान ग्रुपला एक छोटी, गमतीशीर गोष्ट किंवा जोक सांग.",
        },
        {
          "id": "L10",
          "title": "उत्सुकता",
          "desc":
              "कोणाला तरी ते कोणत्या गावचे आहेत आणि त्या गावात त्यांना काय आवडते हे विचार.",
        },
        {
          "id": "L11",
          "title": "खरा रस घेणे",
          "desc":
              "सहकाऱ्याला कामाव्यतिरिक्त असणाऱ्या त्याच्या छंदांबद्दल विचार.",
        },
        {
          "id": "L12",
          "title": "सहज सल्ला",
          "desc":
              "ज्या गोष्टीत तू निष्णात आहेस अशा एका विषयावर कोणालातरी उपयोगी सल्ला दे.",
        },
        {
          "id": "L13",
          "title": "ग्रुपमध्ये सहमती",
          "desc":
              "एका लहान ग्रुप चर्चेत कोणाच्या तरी मताला पाठिंबा देण्यासाठी मान डोलाव.",
        },
        {
          "id": "L14",
          "title": "सहज आमंत्रण",
          "desc":
              "कोणाला तरी 'दुपारच्या जेवणासाठी आमच्यासोबत यायला आवडेल का?' असे विचार.",
        },
        {
          "id": "L15",
          "title": "प्रामाणिक भावना व्यक्त करणे",
          "desc":
              "कोणाला तरी 'तू जेव्हा X केलेस तेव्हा मला खूप आनंद झाला होता' असे सांगून त्याचे कारण स्पष्ट कर.",
        },
        {
          "id": "L16",
          "title": "उत्सुकतेची जागा",
          "desc":
              "कोणाला तरी 'मी नेहमी विचार करतो, एक्स (X) खरं तर कसं काम करतं?' असे विचार.",
        },
        {
          "id": "L17",
          "title": "लहान ग्रुप लीड",
          "desc":
              "एक ग्रुपमध्ये २ किंवा ३ जणांना उत्तर द्यावे लागेल असा एक प्रश्न विचार.",
        },
        {
          "id": "L18",
          "title": "खरे कौतुक",
          "desc":
              "कोणाच्या तरी स्वभावगुणाचे कौतुक कर (उदा: 'तू खूप चांगला ऐकून घेतोस').",
        },
        {
          "id": "L19",
          "title": "समान अनुभव",
          "desc":
              "संभाषणादरम्यान 'मी सुद्धा त्या परिस्थितीत होतो' असे सांगून जोडला जा.",
        },
        {
          "id": "L20",
          "title": "अर्थपूर्ण शांतता",
          "desc":
              "संभाषणात शांततेचा क्षण आल्यास तो लगेच शब्दांनी भरण्याचा प्रयत्न न करता सहज राहू दे.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "धैर्याची सुरुवात",
          "desc":
              "तुला जास्त माहीत नसलेल्या एका व्यक्तीशी संभाषणाला सुरुवात कर.",
        },
        {
          "id": "ST2",
          "title": "प्रामाणिक सहभाग",
          "desc": "एका ग्रुपमध्ये एक छोटी वैयक्तिक गोष्ट किंवा मत शेअर कर.",
        },
        {
          "id": "ST3",
          "title": "चर्चा करणे",
          "desc":
              "कोणाच्या तरी मताशी मर्यादापूर्वक असहमती दर्शव आणि त्याचे कारण स्पष्ट कर.",
        },
        {
          "id": "ST4",
          "title": "ग्रुपमध्ये सामील होणे",
          "desc":
              "चालू असलेल्या ग्रुप संभाषणात सामील हो आणि एक विचारपूर्वक वाक्य जोड.",
        },
        {
          "id": "ST5",
          "title": "टॉपिक सुरू करणे",
          "desc": "एका सामाजिक समूहात चर्चेसाठी एक नवीन विषय मांड.",
        },
        {
          "id": "ST6",
          "title": "सभेत प्रश्न विचारणे",
          "desc":
              "एका जाहीर सभेत किंवा क्लासरूमच्या वातावरणात एक प्रश्न विचार.",
        },
        {
          "id": "ST7",
          "title": "हिंमतीची विनंती",
          "desc":
              "एका कॅफे किंवा पार्कमध्ये एका अनोळखी व्यक्तीला त्याच्या शेजारी बसू शकतो का, असे विचार.",
        },
        {
          "id": "ST8",
          "title": "संवादाचा पूल",
          "desc":
              "एकमेकांना माहीत नसलेल्या दोघांची ओळख करून दे आणि त्यांच्यात एक समान धागा शोधून काढ.",
        },
        {
          "id": "ST9",
          "title": "ठाम गरज व्यक्त करणे",
          "desc":
              "तुला त्रास देणारी एखादी गोष्ट थांबवण्यास किंवा कोणालातरी बाजूला होण्यास मरण्यपूर्वक सांग.",
        },
        {
          "id": "ST10",
          "title": "गोष्टी सांगणारा",
          "desc":
              "३ किंवा अधिक लोक असलेल्या ग्रुपला एक गोष्ट सांगण्यात पुढाकार घे.",
        },
        {
          "id": "ST11",
          "title": "आव्हानात्मक विचार मांडणे",
          "desc":
              "ग्रुपमधील एखाद्या सामान्य समजाला एका मैत्रिपूर्ण आणि आदरयुक्त मार्गाने आव्हान दे.",
        },
        {
          "id": "ST12",
          "title": "सामाजिक पुढाकार",
          "desc":
              "एका खोलीत प्रवेश करताना सर्वांना आधी 'नमस्कार' म्हणणारा व्यक्ती तू स्वतः हो.",
        },
        {
          "id": "ST13",
          "title": "सहानुभूतीने ऐकणे",
          "desc":
              "कोणी आपल्या मनातील दुःख किंवा राग सांगत असताना तो ऐक आणि त्याला आधार देणारे उत्तर दे.",
        },
        {
          "id": "ST14",
          "title": "सार्वजनिक सादरीकरण",
          "desc":
              "एका सामाजिक गेट-टुगेदरमध्ये तुझ्या आवडीच्या एका विषयावर १-२ मिनिटे बोल.",
        },
        {
          "id": "ST15",
          "title": "कमकुवतपणा मान्य करणे",
          "desc":
              "एका ग्रुप समोर तू एखाद्या गोष्टीत नर्वस होतास हे मान्य कर आणि त्यावर सगळे मिळून हसा.",
        },
        {
          "id": "ST16",
          "title": "मर्यादा आखणे",
          "desc":
              "कोणतेही मोठे स्पष्टीकरण न देता, तुला जायचे नसलेले आमंत्रण मरण्यपूर्वक नाकार.",
        },
        {
          "id": "ST17",
          "title": "सक्रिय मध्यस्थ",
          "desc":
              "एका मतभेदामध्ये दोन व्यक्तींना एका मध्यस्थ निर्णयावर आणण्यासाठी मदत कर.",
        },
        {
          "id": "ST18",
          "title": "जाहीर कौतुक",
          "desc":
              "एका ग्रुपमध्ये कोणाच्या तरी प्रयत्नांचे किंवा मिळवलेल्या यशाचे जाहीर कौतुक कर.",
        },
        {
          "id": "ST19",
          "title": "थेट दृष्टिकोन",
          "desc":
              "तुला आवश्यक असलेल्या एका मदतीसाठी किंवा सल्ल्यासाठी कोणालातरी थेट विनंती कर.",
        },
        {
          "id": "ST20",
          "title": "संभाषणाला कलाटणी",
          "desc": "एक संभाषणाला बोरिंग टॉपिकवरून एका रंजक विषयाकडे हळूच वळव.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "भेटवस्तू",
          "desc":
              "कोणाला तरी एक लहान गिफ्ट किंवा खाण्याची वस्तू देऊन 'तू हे आवडेल असे वाटले' म्हणून सांग.",
        },
        {
          "id": "B2",
          "title": "हिंमतीचे नेतृत्व",
          "desc":
              "एका लहान ग्रुपच्या लोकांना एक प्लॅन किंवा एखादी जागा बघण्याची कल्पना सुचव.",
        },
        {
          "id": "B3",
          "title": "मूल्य व्यक्त करणे",
          "desc":
              "तुझ्या आयुष्यात त्यांच्या असण्याचे मूल्य तू का महत्त्वाचे मानतोस, हे एकाला विशेष रीतीने सांग.",
        },
        {
          "id": "B4",
          "title": "सामाजिक संयोजक",
          "desc":
              "काही लोकांसाठी एक छोटी मिटिंग किंवा एका कॉफी डेटचे आयोजन कर.",
        },
        {
          "id": "B5",
          "title": "खोलवर संवाद",
          "desc":
              "कोणाशी तरी १५ मिनिटांपेक्षा जास्त वेळ सखोल आणि अर्थपूर्ण संवाद साध.",
        },
        {
          "id": "B6",
          "title": "आत्मविश्वासाचे शिखर",
          "desc":
              "तुला बघून भीती किंवा संकोच वाटेल अशा एका व्यक्तीशी संभाषणाला सुरुवात कर.",
        },
        {
          "id": "B7",
          "title": "जाहीर शुभेच्छा",
          "desc":
              "एका ग्रुपमध्ये एका ठराविक व्यक्तीसाठी एक लहान, सकारात्मक कौतुकाचे वाक्य सांग किंवा त्याच्या प्रयत्नांचे स्वागत कर.",
        },
        {
          "id": "B8",
          "title": "मर्यादा आखणारा",
          "desc":
              "कोणत्याही मोठ्या स्पष्टीकरणाशिवाय, एका अगतिक मागणीला ठामपणे पण मऊपणाने 'नाही' म्हण.",
        },
        {
          "id": "B9",
          "title": "थेट मागणी",
          "desc":
              "तू मानत असलेल्या किंवा आदर करत असलेल्या कोणाशी तरी १० मिनिटे बोलण्यासाठी किंवा मार्गदर्शनासाठी विनंती कर.",
        },
        {
          "id": "B10",
          "title": "भावनिक प्रयत्न",
          "desc":
              "तुझ्या मित्राशी भावना किंवा मानसिक आरोग्याबद्दल सखोल चर्चा सुरू कर.",
        },
        {
          "id": "B11",
          "title": "सामाजिक मध्यस्थ",
          "desc":
              "शांत संभाषणाद्वारे दोन व्यक्तींमधील एक लहान वाद मिटवण्यासाठी मदत कर.",
        },
        {
          "id": "B12",
          "title": "धैर्याचे कौतुक",
          "desc":
              "पूर्णपणे अनोळखी असलेल्या एका व्यक्तीला तू खरंच त्याच्यामध्ये स्तुत्य वाटणारी एक गोष्ट सांग.",
        },
        {
          "id": "B13",
          "title": "नेटवर्किंग पाऊल",
          "desc":
              "तुझ्या क्षेत्रातील एका तज्ज्ञ किंवा पात्र व्यक्तीशी स्वतःची ओळख करून दे आणि सल्ला माग.",
        },
        {
          "id": "B14",
          "title": "हिंमतीचे सत्य",
          "desc":
              "कोणाशी तरी सांगायला कठीण पण नात्यासाठी फायदेशीर असलेले एक खरे सत्य बोल.",
        },
        {
          "id": "B15",
          "title": "पूर्ण बहर",
          "desc":
              "एक लहान सामाजिक कार्यक्रम आयोजित कर आणि तिथे येणारा प्रत्येक अतिथी कम्फर्टेबल राहील याची काळजी घे.",
        },
        {
          "id": "B16",
          "title": "सार्वजनिक वक्ता",
          "desc":
              "एका मिटिंग किंवा इव्हेंटचा एक छोटा भाग लीड करण्यासाठी किंवा बोलण्यासाठी स्वतःहून पुढे ये.",
        },
        {
          "id": "B17",
          "title": "अनुभवांचा मार्गदर्शक",
          "desc":
              "दुसऱ्याला प्रोत्साहन देण्यासाठी तुझ्या कठीण परिस्थितीचा किंवा तू पार केलेल्या अपयशाचा अनुभव शेअर कर.",
        },
        {
          "id": "B18",
          "title": "धैर्याची माफी",
          "desc":
              "भूतकाळातील एका चुकीसाठी माफी मागण्यासाठी तू स्वतःहून संभाषणाची सुरुवात कर, मग ते कितीही जुने असले तरी चालेल.",
        },
        {
          "id": "B19",
          "title": "मार्गदर्शक (Mentor)",
          "desc":
              "तुझ्यापेक्षा अनुभवाने कमी असलेल्या एका व्यक्तीला एखादी स्किल किंवा कामात मदत करण्यासाठी पुढे राहा.",
        },
        {
          "id": "B20",
          "title": "सामाजिक शिल्पकार",
          "desc":
              "मित्र परिवारासाठी एक नवीन सामाजिक परंपरा किंवा वारंवार होणाऱ्या एका गेट-टुगेदरची (Meetup) सुरुवात कर.",
        },
      ],
    },
    'pt': {
      "Seedling": [
        {
          "id": "S1",
          "title": "O Primeiro Passo",
          "desc": "Faz contacto visual e sorri para uma pessoa hoje.",
        },
        {
          "id": "S2",
          "title": "Um Olá Simples",
          "desc": "Diz 'Bom dia' ou 'Olá' a um vizinho.",
        },
        {
          "id": "S3",
          "title": "O Agradecimento",
          "desc": "Diz 'Obrigado' de forma clara a um funcionário de uma loja.",
        },
        {
          "id": "S4",
          "title": "A Observação",
          "desc": "Nota algo positivo num desconhecido e sorri.",
        },
        {
          "id": "S5",
          "title": "O Aceno Discreto",
          "desc": "Acena para alguém que reconheças à distância.",
        },
        {
          "id": "S6",
          "title": "Segurar a Porta",
          "desc": "Segura a porta aberta para a pessoa que vem atrás de ti.",
        },
        {
          "id": "S7",
          "title": "O Aceno com a Cabeça",
          "desc":
              "Faz um aceno amigável com a cabeça a um colega ao passares por ele.",
        },
        {
          "id": "S8",
          "title": "O Espelho",
          "desc":
              "Pratica o teu 'sorriso confiante' em frente ao espelho durante 1 minuto.",
        },
        {
          "id": "S9",
          "title": "O Olhar Breve",
          "desc":
              "Olha para alguém durante 2 segundos, depois sorri e desvia o olhar.",
        },
        {
          "id": "S10",
          "title": "O Elogio Silencioso",
          "desc":
              "Escreve um comentário simpático na publicação de alguém nas redes sociais.",
        },
        {
          "id": "S11",
          "title": "Partilhar o Espaço",
          "desc":
              "Senta-te ao lado de alguém numa área pública sem desviares o olhar imediatamente.",
        },
        {
          "id": "S12",
          "title": "O Reconhecimento Simples",
          "desc":
              "Diz 'Com licença' ou 'Desculpe' educadamente ao passar por alguém num corredor.",
        },
        {
          "id": "S13",
          "title": "A Saudação Calorosa",
          "desc": "Diz 'Olá' a um estafeta ou entregador.",
        },
        {
          "id": "S14",
          "title": "O Pequeno Aceno",
          "desc":
              "Acena para uma criança ou um animal de estimação (com a permissão do dono).",
        },
        {
          "id": "S15",
          "title": "O Sorriso Suave",
          "desc": "Sorri para três pessoas diferentes hoje.",
        },
        {
          "id": "S16",
          "title": "O Desafio do Contacto Visual",
          "desc":
              "Mantém contacto visual com um operador de caixa até que ele desvie o olhar primeiro.",
        },
        {
          "id": "S17",
          "title": "A Respiração Calma",
          "desc":
              "Dá 3 respirações profundas antes de entrares num espaço social hoje.",
        },
        {
          "id": "S18",
          "title": "A Presença",
          "desc":
              "Fica numa área movimentada durante 5 minutos sem olhares para o teu telemóvel.",
        },
        {
          "id": "S19",
          "title": "O Aceno Casual",
          "desc":
              "Acena com a cabeça para um desconhecido que faça contacto visual contigo.",
        },
        {
          "id": "S20",
          "title": "A Voz Suave",
          "desc": "Diz 'Tenha um bom dia' a alguém ao saíres de uma loja.",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "O Elogio",
          "desc": "Faz um elogio sincero a um colega de trabalho ou de turma.",
        },
        {
          "id": "SP2",
          "title": "A Pergunta",
          "desc": "Pede as horas ou direções a um desconhecido.",
        },
        {
          "id": "SP3",
          "title": "Conversa Fiada",
          "desc":
              "Pergunta a alguém 'Como está a correr o teu dia?' e escuta a resposta.",
        },
        {
          "id": "SP4",
          "title": "O Pedido",
          "desc":
              "Pede ajuda a um funcionário de uma loja para encontrares um item específico.",
        },
        {
          "id": "SP5",
          "title": "O Pedido no Balcão",
          "desc":
              "Pede uma bebida ou comida e pergunta aos funcionários como estão.",
        },
        {
          "id": "SP6",
          "title": "A Apresentação",
          "desc": "Apresenta-te a alguém novo na tua área.",
        },
        {
          "id": "SP7",
          "title": "Conversa sobre o Tempo",
          "desc":
              "Menciona o estado do tempo a alguém enquanto esperas numa fila.",
        },
        {
          "id": "SP8",
          "title": "A Pergunta Simples",
          "desc": "Pergunta a um colega 'O que fizeste no fim de semana?'",
        },
        {
          "id": "SP9",
          "title": "A Oferta de Ajuda",
          "desc":
              "Pergunta a alguém 'Precisas de ajuda com isso?' se parecer que está a passar dificuldades.",
        },
        {
          "id": "SP10",
          "title": "A Opinião",
          "desc":
              "Pergunta a um amigo 'O que achas disto?' sobre um objeto pequeno.",
        },
        {
          "id": "SP11",
          "title": "A Confirmação",
          "desc":
              "Confirma um detalhe com um desconhecido (ex: 'Esta é a fila certa?').",
        },
        {
          "id": "SP12",
          "title": "O Espaço Partilhado",
          "desc":
              "Faz um pequeno comentário sobre o ambiente (ex: 'Está mesmo muita gente aqui').",
        },
        {
          "id": "SP13",
          "title": "O Pequeno Favor",
          "desc":
              "Pede a alguém para te passar algo (como um guardanapo) à mesa.",
        },
        {
          "id": "SP14",
          "title": "O Feedback Caloroso",
          "desc":
              "Diz a um empregado de mesa que a comida estava excelente antes de saíres.",
        },
        {
          "id": "SP15",
          "title": "O Contacto Casual",
          "desc":
              "Envia uma mensagem de 'Como estás?' para alguém com quem já não falas há um mês.",
        },
        {
          "id": "SP16",
          "title": "A Pergunta Aberta",
          "desc":
              "Pergunta a alguém 'Qual é o teu lugar favorito para visitar nesta cidade?'",
        },
        {
          "id": "SP17",
          "title": "O Risco Mínimo",
          "desc":
              "Pergunta a um desconhecido se sabe onde fica a casa de banho mais próxima.",
        },
        {
          "id": "SP18",
          "title": "Elogiar um Objeto",
          "desc": "Diz a alguém que gostas dos sapatos/mala/acessório dela.",
        },
        {
          "id": "SP19",
          "title": "A Pausa Educada",
          "desc":
              "Espera que a outra pessoa termine completamente de falar antes de responderes.",
        },
        {
          "id": "SP20",
          "title": "O Aceno Amigável",
          "desc":
              "Acena e diz 'Adeus' a alguém com quem acabaste de ter uma breve interação.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "Buscador de Opiniões",
          "desc":
              "Pede a alguém a sua opinião sobre um livro, filme ou música.",
        },
        {
          "id": "L2",
          "title": "O Detalhe",
          "desc":
              "Faz uma pergunta de acompanhamento depois de alguém te contar algo sobre si.",
        },
        {
          "id": "L3",
          "title": "A Recomendação",
          "desc":
              "Pede a um desconhecido uma recomendação de um bom sítio para comer por perto.",
        },
        {
          "id": "L4",
          "title": "A Conexão",
          "desc":
              "Encontra um interesse comum com alguém e fala sobre isso durante 2 minutos.",
        },
        {
          "id": "L5",
          "title": "A Mão Amiga",
          "desc":
              "Oferece-te para ajudar alguém com uma pequena tarefa (como carregar um saco).",
        },
        {
          "id": "L6",
          "title": "A Observação Social",
          "desc":
              "Inicia uma conversa com base em algo que esteja a acontecer ao redor de ambos.",
        },
        {
          "id": "L7",
          "title": "A Pergunta de Resposta Aberta",
          "desc": "Pergunta a alguém 'Como começaste a trabalhar nesta área?'",
        },
        {
          "id": "L8",
          "title": "O Ouvinte Ativo",
          "desc":
              "Ouve alguém durante 3 minutos sem interromper, e depois resume o que a pessoa disse.",
        },
        {
          "id": "L9",
          "title": "O Riso Partilhado",
          "desc":
              "Conta uma história curta e engraçada ou uma piada a um pequeno grupo.",
        },
        {
          "id": "L10",
          "title": "A Curiosidade",
          "desc": "Pergunta a alguém de onde é e o que mais gosta nesse lugar.",
        },
        {
          "id": "L11",
          "title": "O Interesse Sincero",
          "desc":
              "Pergunta a um colega sobre os seus passatempos fora do trabalho.",
        },
        {
          "id": "L12",
          "title": "O Conselho Suave",
          "desc": "Dá a alguém uma dica útil sobre algo em que sejas bom.",
        },
        {
          "id": "L13",
          "title": "O Aceno do Grupo",
          "desc":
              "Concorda com o ponto de vista de alguém numa discussão em um pequeno grupo.",
        },
        {
          "id": "L14",
          "title": "O Convite Casual",
          "desc":
              "Pergunta a alguém 'Gostarias de te juntar a nós para o almoço?'",
        },
        {
          "id": "L15",
          "title": "A Reflexão Honesta",
          "desc":
              "Diz a alguém 'Apreciei imenso quando fizeste X' e explica o porquê.",
        },
        {
          "id": "L16",
          "title": "O Espaço para a Curiosidade",
          "desc":
              "Pergunta a alguém 'Sempre tive curiosidade, como é que X funciona realmente?'",
        },
        {
          "id": "L17",
          "title": "A Liderança do Pequeno Grupo",
          "desc":
              "Faz uma pergunta que exija a resposta de 2 ou 3 pessoas num grupo.",
        },
        {
          "id": "L18",
          "title": "O Elogio Sincero",
          "desc":
              "Elogia alguém por um traço de personalidade (ex: 'És um ótimo ouvinte').",
        },
        {
          "id": "L19",
          "title": "A Experiência Partilhada",
          "desc":
              "Diz 'Eu também já estive nessa situação' durante uma conversa.",
        },
        {
          "id": "L20",
          "title": "A Pausa Significativa",
          "desc":
              "Permite que ocorra um momento de silêncio numa conversa sem correres para o preencher.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "O Início Corajoso",
          "desc": "Inicia uma conversa com alguém que não conheces bem.",
        },
        {
          "id": "ST2",
          "title": "A Partilha Honesta",
          "desc":
              "Partilha uma pequena história pessoal ou opinião num ambiente de grupo.",
        },
        {
          "id": "ST3",
          "title": "O Debate",
          "desc":
              "Discorda educadamente da opinião de alguém e explica o porquê.",
        },
        {
          "id": "ST4",
          "title": "A Entrada no Grupo",
          "desc":
              "Junta-te a uma conversa de grupo e contribui com uma frase ponderada.",
        },
        {
          "id": "ST5",
          "title": "A Condução do Tema",
          "desc": "Introduz um novo tema de conversa num grupo social.",
        },
        {
          "id": "ST6",
          "title": "A Pergunta Pública",
          "desc":
              "Faz uma pergunta numa reunião pública ou num ambiente de sala de aula.",
        },
        {
          "id": "ST7",
          "title": "O Pedido Ousado",
          "desc":
              "Pergunta a um desconhecido se te podes sentar ao lado dele num café ou parque.",
        },
        {
          "id": "ST8",
          "title": "A Ponte de Conversação",
          "desc":
              "Apresenta duas pessoas que não se conhecem e encontra um ponto comum.",
        },
        {
          "id": "ST9",
          "title": "A Necessidade Assertiva",
          "desc":
              "Pede educadamente a alguém para se mover ou parar de fazer algo que te incomoda.",
        },
        {
          "id": "ST10",
          "title": "O Contador de Histórias",
          "desc":
              "Toma a iniciativa de contar uma história a um grupo de 3 ou mais pessoas.",
        },
        {
          "id": "ST11",
          "title": "O Desafio Aberto",
          "desc":
              "Contesta uma opinião comum num grupo de forma amigável e respeitosa.",
        },
        {
          "id": "ST12",
          "title": "A Iniciativa Social",
          "desc":
              "Sê la primeira pessoa a dizer 'Olá' a todos ao entrar numa sala.",
        },
        {
          "id": "ST13",
          "title": "A Escuta Empática",
          "desc":
              "Ouve alguém que se esteja a desabafar e oferece uma resposta de apoio.",
        },
        {
          "id": "ST14",
          "title": "A Apresentação Pública",
          "desc":
              "Fala durante 1-2 minutos sobre um tema que adores num encontro social.",
        },
        {
          "id": "ST15",
          "title": "A Partilha Vulnerável",
          "desc":
              "Admite perante um grupo que estavas nervoso com alguma coisa, e riam-se disso juntos.",
        },
        {
          "id": "ST16",
          "title": "A Definição de Limites",
          "desc":
              "Recusa educadamente um convite ao qual não queiras comparecer sem dares explicações excessivas.",
        },
        {
          "id": "ST17",
          "title": "O Mediador Ativo",
          "desc":
              "Ajuda duas pessoas a encontrarem um meio-termo num desacordo.",
        },
        {
          "id": "ST18",
          "title": "O Elogio Público",
          "desc":
              "Elogia publicamente o esforço ou conquista de alguém num grupo.",
        },
        {
          "id": "ST19",
          "title": "A Abordagem Direta",
          "desc":
              "Pede diretamente a alguém um favor ou um conselho de que precises.",
        },
        {
          "id": "ST20",
          "title": "O Desvio da Conversa",
          "desc":
              "Transita suavemente uma conversa de um assunto aborrecido para um assunto interessante.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "O Presente",
          "desc":
              "Dá um pequeno agrado a alguém e diz 'Achei que irias gostar disto'.",
        },
        {
          "id": "B2",
          "title": "A Condução Ousada",
          "desc":
              "Sugere um plano ou um lugar para visitar a um pequeno grupo de pessoas.",
        },
        {
          "id": "B3",
          "title": "A Apreciação",
          "desc":
              "Diz a alguém especificamente o motivo pelo qual aprecias tê-lo na tua vida.",
        },
        {
          "id": "B4",
          "title": "O Anfitrião Social",
          "desc":
              "Organiza um pequeno encontro ou um café para algumas pessoas.",
        },
        {
          "id": "B5",
          "title": "O Mergulho Profundo",
          "desc":
              "Tem uma conversa profunda e significativa com alguém por mais de 15 minutos.",
        },
        {
          "id": "B6",
          "title": "O Pico de Confiança",
          "desc": "Inicia uma conversa com alguém que aches intimidador.",
        },
        {
          "id": "B7",
          "title": "O Brindis Público",
          "desc":
              "Faz um brinde curto e positivo ou um elogio público a alguém num grupo.",
        },
        {
          "id": "B8",
          "title": "O Definitório de Limites",
          "desc":
              "Diz 'Não' a um pedido de forma firme mas gentil, sem dares explicações excessivas.",
        },
        {
          "id": "B9",
          "title": "O Pedido Direto",
          "desc":
              "Pede a alguém que admiras uma conversa de 10 minutos ou mentoria.",
        },
        {
          "id": "B10",
          "title": "A Condução Emocional",
          "desc":
              "Inicia uma conversa sobre sentimentos ou saúde mental com um amigo.",
        },
        {
          "id": "B11",
          "title": "O Mediador Social",
          "desc":
              "Ajuda duas pessoas a resolver um pequeno conflito através de uma conversa calma.",
        },
        {
          "id": "B12",
          "title": "O Elogio Ousado",
          "desc":
              "Diz a um perfeito desconhecido algo que admiras genuinamente nele.",
        },
        {
          "id": "B13",
          "title": "O Passo de Networking",
          "desc":
              "Apresenta-te a um profissional da tua área e pede conselhos.",
        },
        {
          "id": "B14",
          "title": "A Verdade Corajosa",
          "desc":
              "Diz a alguém uma verdade que seja difícil, mas útil para o vosso relacionamento.",
        },
        {
          "id": "B15",
          "title": "O Pleno Desabrochar",
          "desc":
              "Organiza um pequeno evento social e certifica-te de que cada convidado se sente bem-vindo.",
        },
        {
          "id": "B16",
          "title": "O Orador Público",
          "desc":
              "Oferece-te como voluntário para falar ou liderar uma pequena parte de uma reunião ou evento.",
        },
        {
          "id": "B17",
          "title": "A Condução Vulnerável",
          "desc":
              "Partilha uma dificuldade que tenhas superado para encorajar outra pessoa.",
        },
        {
          "id": "B18",
          "title": "As Desculpas Ousadas",
          "desc":
              "Inicia uma conversa para pedir desculpas por um erro passado, mesmo que tenha sido há muito tempo.",
        },
        {
          "id": "B19",
          "title": "O Mentor",
          "desc":
              "Oferece-te para ajudar alguém com menos experiência do que tu numa determinada competência.",
        },
        {
          "id": "B20",
          "title": "O Arquiteto Social",
          "desc":
              "Cria uma nova tradição social ou um encontro recorrente para um grupo de amigos.",
        },
      ],
    },
    'no': {
      "Seedling": [
        {
          "id": "S1",
          "title": "Det første skrittet",
          "desc": "Opprett øyekontakt og smil til én person i dag.",
        },
        {
          "id": "S2",
          "title": "Et enkelt hei",
          "desc": "Si 'God morgen' eller 'Hei' til en nabo.",
        },
        {
          "id": "S3",
          "title": "Takket",
          "desc": "Si 'Takk' tydelig til en butikkansatt.",
        },
        {
          "id": "S4",
          "title": "Observasjonen",
          "desc": "Legg merke til noe positivt ved en fremmed og smil.",
        },
        {
          "id": "S5",
          "title": "Det stille vinket",
          "desc": "Vink til noen du kjenner igjen på avstand.",
        },
        {
          "id": "S6",
          "title": "Hold døren",
          "desc": "Hold døren åpen for noen bak deg.",
        },
        {
          "id": "S7",
          "title": "Nikket",
          "desc": "Gi et vennlig nikk til en kollega når du går forbi.",
        },
        {
          "id": "S8",
          "title": "Speilet",
          "desc": "Øv på ditt 'selvsikre smil' i speilet i 1 minutt.",
        },
        {
          "id": "S9",
          "title": "Det korte blikket",
          "desc": "Se på noen i 2 sekunder, smil og se bort.",
        },
        {
          "id": "S10",
          "title": "Den stille rosen",
          "desc": "Skriv en hyggelig kommentar på sosiale medier til noen.",
        },
        {
          "id": "S11",
          "title": "Dele plassen",
          "desc":
              "Sett deg ved siden av noen på et offentlig sted uten å se bort med en gang.",
        },
        {
          "id": "S12",
          "title": "Enkel høflighet",
          "desc":
              "Si 'Unnskyld meg' på en høflig måte når du går forbi noen i en korridor.",
        },
        {
          "id": "S13",
          "title": "Den varme hilsenen",
          "desc": "Si 'Hei' til et bud eller en sjåfør.",
        },
        {
          "id": "S14",
          "title": "Det lille vinket",
          "desc":
              "Vink til et barn eller et kjæledyr (med eierens tillatelse).",
        },
        {
          "id": "S15",
          "title": "Det myke smilet",
          "desc": "Smil til tre forskjellige personer i dag.",
        },
        {
          "id": "S16",
          "title": "Øyekontakt-utfordringen",
          "desc":
              "Hold øyekontakt med en kassamedarbeider til de ser bort først.",
        },
        {
          "id": "S17",
          "title": "Det rolige pustet",
          "desc": "Ta 3 dype åndedrag før du går inn i et sosialt rom i dag.",
        },
        {
          "id": "S18",
          "title": "Tilstedeværelsen",
          "desc":
              "Stå i et overfylt område i 5 minutter uten å se på telefonen din.",
        },
        {
          "id": "S19",
          "title": "Det uformelle nikket",
          "desc": "Nikk til en fremmed som oppretter øyekontakt med deg.",
        },
        {
          "id": "S20",
          "title": "Den milde stemmen",
          "desc": "Si 'Ha en fin dag' til noen når du forlater en butikk.",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "Komplimentet",
          "desc": "Gi et oppriktig kompliment til en kollega eller medstudent.",
        },
        {
          "id": "SP2",
          "title": "Spørsmålet",
          "desc": "Spør en fremmed om klokka eller veibeskrivelse.",
        },
        {
          "id": "SP3",
          "title": "Småprat",
          "desc": "Spør noen 'Hvordan går dagen din?' og lytt til svaret.",
        },
        {
          "id": "SP4",
          "title": "Henvendelsen",
          "desc": "Spør en butikkansatt om hjelp til å finne en bestemt vare.",
        },
        {
          "id": "SP5",
          "title": "Bestillingen",
          "desc":
              "Bestill en drink eller mat og spør de ansatte hvordan de har det.",
        },
        {
          "id": "SP6",
          "title": "Hilsenen",
          "desc": "Introduser deg selv for noen ny i nærområdet ditt.",
        },
        {
          "id": "SP7",
          "title": "Værpraten",
          "desc": "Nevn været til noen mens du venter i en kø.",
        },
        {
          "id": "SP8",
          "title": "Det enkle spørsmålet",
          "desc": "Spør en kollega 'Hva gjorde du i helgen?'",
        },
        {
          "id": "SP9",
          "title": "Hjelpetilbudet",
          "desc":
              "Spør noen 'Trenger du hjelp med det?' hvis de ser ut til å streve.",
        },
        {
          "id": "SP10",
          "title": "Meningen",
          "desc":
              "Spør en venn 'Hva synes du om denne?' om en liten gjenstand.",
        },
        {
          "id": "SP11",
          "title": "Bekreftelsen",
          "desc":
              "Bekreft en detalj med en fremmed (f.eks. 'Er dette den riktige køen?').",
        },
        {
          "id": "SP12",
          "title": "Det delte rommet",
          "desc":
              "Kom med en liten kommentar om omgivelsene (f.eks. 'Det er veldig fullt her').",
        },
        {
          "id": "SP13",
          "title": "Den lille tjenesten",
          "desc": "Spør noen om å sende deg noe (som en serviett) ved bordet.",
        },
        {
          "id": "SP14",
          "title": "Den varme tilbakemeldingen",
          "desc": "Fortell en servitør at maten var fantastisk før du drar.",
        },
        {
          "id": "SP15",
          "title": "Uformell oppfølging",
          "desc":
              "Send en 'Hvordan går det?'-tekstmelding til noen du ikke har snakket med på en måned.",
        },
        {
          "id": "SP16",
          "title": "Det åpne spørsmålet",
          "desc": "Spør noen 'Hva er ditt favorittsted å besøke i denne byen?'",
        },
        {
          "id": "SP17",
          "title": "Den minste risikoen",
          "desc": "Spør en fremmed om de vet hvor nærmeste toalett er.",
        },
        {
          "id": "SP18",
          "title": "Gjenstandsros",
          "desc": "Fortell noen at du liker skoene/vesken/tilbehøret deres.",
        },
        {
          "id": "SP19",
          "title": "Den høflige pausen",
          "desc": "Vent til noen har snakket helt ferdig før du svarer dem.",
        },
        {
          "id": "SP20",
          "title": "Den vennlige avskjeden",
          "desc":
              "Vink og si 'Ha det' til noen du nettopp hadde en kort interaksjon med.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "Meningstakeren",
          "desc": "Spør noen om deres mening om en bok, film eller sang.",
        },
        {
          "id": "L2",
          "title": "Detaljen",
          "desc":
              "Still et oppfølgingsspørsmål etter at noen har fortalt deg noe om seg selv.",
        },
        {
          "id": "L3",
          "title": "Anbefalingen",
          "desc":
              "Spør en fremmed om en anbefaling til et bra sted å spise i nærheten.",
        },
        {
          "id": "L4",
          "title": "Koblingen",
          "desc":
              "Finn en felles interesse med noen og snakk om det i 2 minutter.",
        },
        {
          "id": "L5",
          "title": "Den hjelpende hånden",
          "desc":
              "Tilby deg å hjelpe noen med en liten oppgave (som å bære en pose).",
        },
        {
          "id": "L6",
          "title": "Sosial observasjon",
          "desc": "Start en samtale basert på noe som skjer rundt dere begge.",
        },
        {
          "id": "L7",
          "title": "Det åpne spørsmålet",
          "desc": "Spør noen 'Hvordan havnet du i dette yrket?'",
        },
        {
          "id": "L8",
          "title": "Den aktive lytteren",
          "desc":
              "Lytt til noen i 3 minutter uten å avbryte, og oppsummer deretter hva de sa.",
        },
        {
          "id": "L9",
          "title": "Den delte lattern",
          "desc":
              "Fortell en kort, morsom historie eller en vits til en liten gruppe.",
        },
        {
          "id": "L10",
          "title": "Nysgjerrigheten",
          "desc": "Spør noen hvor de er fra og hva de liker med det stedet.",
        },
        {
          "id": "L11",
          "title": "Den oppriktige interessen",
          "desc": "Spør en kollega om hobbyene deres utenom jobben.",
        },
        {
          "id": "L12",
          "title": "Det milde rådet",
          "desc": "Gi noen et nyttig tips om noe du er god på.",
        },
        {
          "id": "L13",
          "title": "Gruppe-nikket",
          "desc": "Si deg enig i noens poeng i en liten gruppediskusjon.",
        },
        {
          "id": "L14",
          "title": "Den uformelle invitasjonen",
          "desc": "Spør noen 'Har du lyst til å bli med oss på lunsj?'",
        },
        {
          "id": "L15",
          "title": "Den ærlige refleksjonen",
          "desc":
              "Fortell noen 'Jeg satte stor pris på da du gjorde X' og forklar hvorfor.",
        },
        {
          "id": "L16",
          "title": "Nysgjerrighetsgapet",
          "desc":
              "Spør noen 'Jeg har alltid lurt på, hvordan fungerer egentlig X?'",
        },
        {
          "id": "L17",
          "title": "Mindre gruppeleder",
          "desc":
              "Still et spørsmål som krever at 2 eller 3 personer i en gruppe svarer.",
        },
        {
          "id": "L18",
          "title": "Det ekte komplimentet",
          "desc":
              "Gi noen et kompliment for et personlighetstrekk (f.eks. 'Du er en god lytter').",
        },
        {
          "id": "L19",
          "title": "Den delte opplevelsen",
          "desc":
              "Si 'Jeg har vært i den situasjonen selv' i løpet av en samtale.",
        },
        {
          "id": "L20",
          "title": "Den meningsfulle pausen",
          "desc":
              "Tillat at en stillhet oppstår i en samtale uten å skynde deg med å fylle den.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "Den modige starten",
          "desc": "Start en samtale med noen du ikke kjenner så godt.",
        },
        {
          "id": "ST2",
          "title": "Det ærlige bidraget",
          "desc":
              "Del en liten personlig historie eller mening i en gruppesetting.",
        },
        {
          "id": "ST3",
          "title": "Debatten",
          "desc": "Si deg høflig uenig i noens mening og forklar hvorfor.",
        },
        {
          "id": "ST4",
          "title": "Gruppeinngangen",
          "desc":
              "Bli med i en gruppesamtale og bidra med en gjennomtenkt setning.",
        },
        {
          "id": "ST5",
          "title": "Temalederen",
          "desc": "Introduser et nytt samtaletema i en sosial gruppe.",
        },
        {
          "id": "ST6",
          "title": "Det offentlige spørsmålet",
          "desc":
              "Still et spørsmål i et offentlig møte eller i en klasseromssetting.",
        },
        {
          "id": "ST7",
          "title": "Den dristige forespørselen",
          "desc":
              "Spør en fremmed om du kan sitte ved siden av dem på en kafé eller i en park.",
        },
        {
          "id": "ST8",
          "title": "Samtalebroen",
          "desc":
              "Introduser to personer som ikke kjenner hverandre og finn en fellesnevner.",
        },
        {
          "id": "ST9",
          "title": "Det assertive behovet",
          "desc":
              "Be noen høflig om å flytte seg eller slutte å gjøre noe som plager deg.",
        },
        {
          "id": "ST10",
          "title": "Fortelleren",
          "desc":
              "Ta ledelsen i å fortelle en historie til a gruppe på 3 eller flere personer.",
        },
        {
          "id": "ST11",
          "title": "Den åpne utfordringen",
          "desc":
              "Utfordre en vanlig mening i en gruppe på en vennlig og respektfull måte.",
        },
        {
          "id": "ST12",
          "title": "Sosialt initiativ",
          "desc":
              "Vær den første personen til å si 'Hei' til alle når du går inn i et rom.",
        },
        {
          "id": "ST13",
          "title": "Den empatiske lyttingen",
          "desc":
              "Lytt til noen som tømmer hjertet sitt, og gi en støttende respons.",
        },
        {
          "id": "ST14",
          "title": "Den offentlige presentasjonen",
          "desc":
              "Snakk i 1-2 minutter om et tema du elsker i en sosial sammenkomst.",
        },
        {
          "id": "ST15",
          "title": "Det sårbare bidraget",
          "desc":
              "Innrøm overfor en gruppe at du var nervøs for noe, og le av det sammen.",
        },
        {
          "id": "ST16",
          "title": "Sette grenser",
          "desc":
              "Avslå høflig en invitasjon du ikke ønsker å delta på uten å overforklare.",
        },
        {
          "id": "ST17",
          "title": "Den aktive megleren",
          "desc": "Hjelp to personer med å finne en mellomting i en uenighet.",
        },
        {
          "id": "ST18",
          "title": "Det offentlige komplimentet",
          "desc": "Ros offentlig noens innsats eller prestasjon i en gruppe.",
        },
        {
          "id": "ST19",
          "title": "Den direkte tilnærmingen",
          "desc": "Spør noen direkte om en tjeneste eller et råd du trenger.",
        },
        {
          "id": "ST20",
          "title": "Samtalevendingen",
          "desc":
              "Få en samtale til å gå knirkefritt over fra et kjedelig tema til et interessant et.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "Gaven",
          "desc":
              "Gi en liten oppmerksomhet til noen og si 'Jeg tenkte du ville like denne'.",
        },
        {
          "id": "B2",
          "title": "Den modige ledelsen",
          "desc":
              "Foreslå en plan eller et sted å besøke til en liten gruppe mennesker.",
        },
        {
          "id": "B3",
          "title": "Verద్దsettelsen",
          "desc":
              "Fortell noen spesifikt hvorfor du setter pris på å ha dem i livet ditt.",
        },
        {
          "id": "B4",
          "title": "Den sosiale verten",
          "desc":
              "Organiser en liten sammenkomst eller en kaffeavtale for noen få personer.",
        },
        {
          "id": "B5",
          "title": "Dypdykket",
          "desc": "Ha en dyp, meningsfull samtale med noen i over 15 minutter.",
        },
        {
          "id": "B6",
          "title": "Selvtillitstoppen",
          "desc": "Start en samtale med noen du synes er skremmende.",
        },
        {
          "id": "B7",
          "title": "Den offentlige skålen",
          "desc":
              "Hold en kort, positiv skål eller gi en anerkjennelse til noen i en gruppe.",
        },
        {
          "id": "B8",
          "title": "Grensesetteren",
          "desc":
              "Si 'Nei' til en forespørsel på en bestemt, men vennlig måte, uten å overforklare.",
        },
        {
          "id": "B9",
          "title": "Den direkte forespørselen",
          "desc":
              "Spør noen du beundrer om en 10-minutters prat eller mentorskap.",
        },
        {
          "id": "B10",
          "title": "Den følelsesmessige ledelsen",
          "desc":
              "Start en samtale om følelser eller mental helse med en venn.",
        },
        {
          "id": "B11",
          "title": "Sosial megler",
          "desc":
              "Hjelp to personer med å løse en liten konflikt gjennom en rolig samtale.",
        },
        {
          "id": "B12",
          "title": "Det modige komplimentet",
          "desc":
              "Fortell en helt fremmed person noe du oppriktig beundrer ved dem.",
        },
        {
          "id": "B13",
          "title": "Nettverkssteget",
          "desc":
              "Introduser deg selv for en fagperson innen ditt felt og spør om råd.",
        },
        {
          "id": "B14",
          "title": "Den modige sannheten",
          "desc":
              "Fortell noen en sannhet som er vanskelig, men nyttig for forholdet.",
        },
        {
          "id": "B15",
          "title": "Full blomst",
          "desc":
              "Arranger et lite sosialt arrangement og sørg for at hver gjest føler seg velkommen.",
        },
        {
          "id": "B16",
          "title": "Den offentlige taleren",
          "desc":
              "Meld deg frivillig til å snakke eller lede en liten del av et møte eller arrangement.",
        },
        {
          "id": "B17",
          "title": "Sårbar ledelse",
          "desc":
              "Del en utfordring du har overvunnet for å oppmuntre noen andre.",
        },
        {
          "id": "B18",
          "title": "Den modige unnskyldningen",
          "desc":
              "Start en samtale for å si unnskyld for en tidligere feil, selv om det var lenge siden.",
        },
        {
          "id": "B19",
          "title": "Mentoren",
          "desc":
              "Tilby deg å hjelpe noen som er mindre erfaren enn deg med en ferdighet.",
        },
        {
          "id": "B20",
          "title": "Den sosiale arkitekten",
          "desc":
              "Opprett en ny sosial tradisjon eller et tilbakevendende treff for en vennegjerd.",
        },
      ],
    },
    'gu': {
      "Seedling": [
        {
          "id": "S1",
          "title": "પહેલું પગલું",
          "desc": "આજે કોઈ એક વ્યક્તિ સાથે આંખો મિલાવીને તેની સામે જોઇને હસો.",
        },
        {
          "id": "S2",
          "title": "એક સામાન્ય હેલો",
          "desc": "કોઈ પડોશીને 'શુભ સવાર' અથવા 'નમસ્તે' કહો.",
        },
        {
          "id": "S3",
          "title": "આભાર માનવો",
          "desc": "કોઈ દુકાનદારને સ્પષ્ટ અવાજે 'આભાર' કહો.",
        },
        {
          "id": "S4",
          "title": "અવલોકન",
          "desc": "કોઈ અજાણી વ્યક્તિમાં કોઈ સકારાત્મક બાબત નોંધીને હસો.",
        },
        {
          "id": "S5",
          "title": "શાંતિથી હાથ હલાવવો",
          "desc": "દૂરથી તમે ઓળખેલા કોઈ પણ વ્યક્તિને સહેજ હાથ હલાવીને આવકારો.",
        },
        {
          "id": "S6",
          "title": "દરવાજો પકડી રાખવો",
          "desc": "તમારી પાછળ આવતી વ્યક્તિ માટે દરવાજો ખોલીને પકડી રાખો.",
        },
        {
          "id": "S7",
          "title": "માથું નમાવવું",
          "desc":
              "બાજુમાંથી પસાર થતી વખતે કોઈ સહકર્મીને સ્નેહપૂર્વક માથું નમાવીને નમસ્કાર કરો.",
        },
        {
          "id": "S8",
          "title": "અરીસાની પ્રેક્ટિસ",
          "desc":
              "અરીસાની સામે ઊભા રહીને ૧ મિનિટ માટે તમારા 'આત્મવિશ્વાસપૂર્ણ હાસ્ય'ની પ્રેક્ટિસ કરો.",
        },
        {
          "id": "S9",
          "title": "ટૂંકી નજર",
          "desc": "કોઈની સામે ૨ સેકન્ડ જુઓ, પછી સ્મિત આપીને નજર ફેરવી લો.",
        },
        {
          "id": "S10",
          "title": "શાંતિથી પ્રશંસા",
          "desc": "કોઈની સોશિયલ મીડિયા પોસ્ટની નીચે એક સરસ કોમેન્ટ લખો.",
        },
        {
          "id": "S11",
          "title": "જગ્યા વહેંચવી",
          "desc":
              "જાહેર વિસ્તારમાં કોઈની બાજુમાં બેસો અને તરત જ નજર ફેરવ્યા વગર ત્યાં થોડો સમય વિતાવો.",
        },
        {
          "id": "S12",
          "title": "સામાન્ય નમ્રતા",
          "desc":
              "કોરિડોરમાં કોઈની બાજુમાંથી પસાર થતી વખતે નમ્રતાપૂર્વક 'એક્સક્યુઝ મી' કહો.",
        },
        {
          "id": "S13",
          "title": "હૂંફાળું સ્વાગત",
          "desc": "કોઈ ડિલિવરી બોય કે કુરિયર લાવનાર વ્યક્તિને 'નમસ્તે' કહો.",
        },
        {
          "id": "S14",
          "title": "નાનો ઈશારો",
          "desc":
              "કોઈ નાના બાળકને અથવા (માલિકની પરવાનગીથી) પાળતુ પ્રાણીને હાથ હલાવીને બતાવો.",
        },
        {
          "id": "S15",
          "title": "નમ્ર સ્મિત",
          "desc": "આજે ત્રણ અલગ-અલગ લોકો સામે જોઈને હસો.",
        },
        {
          "id": "S16",
          "title": "આઈ કોન્ટેક્ટનું આહ્વાન",
          "desc":
              "કેશિયર પહેલા નજર ફેરવે ત્યાં સુધી તેની સાથે આંખો મિલાવી રાખો.",
        },
        {
          "id": "S17",
          "title": "પ્રશાંત શ્વાસ",
          "desc":
              "આજે કોઈ પણ સામાજિક જગ્યાએ પ્રવેશતા પહેલા ૩ વખત ઊંડા શ્વાસ લો.",
        },
        {
          "id": "S18",
          "title": "હાજરી",
          "desc":
              "ભીડવાળી જગ્યાએ ફોન બિલકુલ જોયા વગર 5 મિનિટ માટે શાંતિથી ઊભા રહો.",
        },
        {
          "id": "S19",
          "title": "સહજતાથી માથું હલાવવું",
          "desc":
              "તમારી સાથે આંખો મિલાવનાર અજાણ્યા વ્યક્તિને માથું હલાવીને પ્રતિસાદ આપો.",
        },
        {
          "id": "S20",
          "title": "સરસ વિદાય",
          "desc":
              "દુકાનમાંથી બહાર નીકળતી વખતે કોઈને 'તમારો દિવસ સારો જાય' તેમ કહો.",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "સાચી પ્રશંસા",
          "desc": "કોઈ સહકર્મી અથવા ક્લાસમેટની મનથી પ્રશંસા કરો.",
        },
        {
          "id": "SP2",
          "title": "પ્રશ્ન પૂછવો",
          "desc": "કોઈ અજાણી વ્યક્તિને સમય અથવા રસ્તો પૂછો.",
        },
        {
          "id": "SP3",
          "title": "નાની ચર્ચા",
          "desc":
              "કોઈને 'આજનો દિવસ કેવો ચાલી રહ્યો છે?' તેમ પૂછો અને તેનો જવાબ બરાબર સાંભળો.",
        },
        {
          "id": "SP4",
          "title": "મદદ માંગવી",
          "desc": "કોઈ ચોક્કસ વસ્તુ શોધવા માટે સ્ટોરના કર્મચારીની મદદ માગો.",
        },
        {
          "id": "SP5",
          "title": "ઓર્ડર આપતી વખતે",
          "desc":
              "કોઈ ડ્રિંક કે ફૂડ ઓર્ડર કરો અને ત્યાંના સ્ટાફને તેઓ કેમ છે તે પૂછો.",
        },
        {
          "id": "SP6",
          "title": "ઓળખાણ કરાવવી",
          "desc": "તમારા વિસ્તારની કોઈ નવી વ્યક્તિને તમારી પોતાની ઓળખાણ કરાવો.",
        },
        {
          "id": "SP7",
          "title": "હવામાન વિશે વાતો",
          "desc":
              "લાઈનમાં રાહ જોતી વખતે બાજુવાળી વ્યક્તિ સાથે હવામાન વિશે વાત કરો.",
        },
        {
          "id": "SP8",
          "title": "સરળ પૂછપરછ",
          "desc": "સહકર્મીને પૂછો કે 'તમે વિકેન્ડમાં શું કર્યું?'",
        },
        {
          "id": "SP9",
          "title": "મદદની દરખાસ્ત",
          "desc":
              "જો કોઈ મુશ્કેલીમાં હોય તેવું લાગે, તો તેને પૂછો કે 'તમારે આમાં કોઈ મદદની જરૂર છે?'",
        },
        {
          "id": "SP10",
          "title": "અભિપ્રાય લેવો",
          "desc":
              "કોઈ નાની વસ્તુ બતાવતા મિત્રને પૂછો કે 'તમને આના વિશે શું લાગે છે?'",
        },
        {
          "id": "SP11",
          "title": "ખાતરી કરવી",
          "desc":
              "કોઈ અજાણી વ્યક્તિ સાથે વિગત કન્ફર્મ કરી લો (દા.ત: 'શું આ જ લાઇન બરાબર છે ને?').",
        },
        {
          "id": "SP12",
          "title": "પરિસંવાદ",
          "desc":
              "આસપાસના વાતાવરણ વિશે એક નાની કોમેન્ટ કરો (દા.ત: 'અહીં ખરેખર ખૂબ જ ભીડ છે').",
        },
        {
          "id": "SP13",
          "title": "નાની વિનંતી",
          "desc":
              "ટેબલ પર હોવ ત્યારે કોઈ વસ્તુ (જેમ કે નેપકિન) તમારી તરફ સરકાવવા માટે કોઈને કહો.",
        },
        {
          "id": "SP14",
          "title": "ઉત્કૃષ્ટ ફીડબેક",
          "desc": "નીકળતા પહેલા વેઇટરને કહો કે જમવાનું ખૂબ જ અદ્ભુત હતું.",
        },
        {
          "id": "SP15",
          "title": "સહજ વિચારપૂછ",
          "desc":
              "એક મહિનાથી વાત ન કરી હોય તેવી વ્યક્તિને 'કેમ છે?' એવો એક મેસેજ મોકલો.",
        },
        {
          "id": "SP16",
          "title": "ઓપન ક્વેશ્ચન",
          "desc":
              "કોઈને 'આ શહેરમાં તમારી સૌથી મનપસંદ ફરવાની જગ્યા કઈ છે?' તેમ પૂછો.",
        },
        {
          "id": "SP17",
          "title": "સૌથી નાનું જોખમ",
          "desc":
              "નજીકમાં વૉશરૂમ ક્યાં છે તે ખબર છે કે કેમ, તે કોઈ અજાણી વ્યક્તિને પૂછો.",
        },
        {
          "id": "SP18",
          "title": "વસ્તુની સ્તુતિ",
          "desc": "કોઈને કહો કે તમને તેના શૂઝ/બેગ/એક્સેસરીઝ ગમ્યા.",
        },
        {
          "id": "SP19",
          "title": "નમ્રતાપૂર્વક રાહ જોવી",
          "desc":
              "કોઈને પણ જવાબ આપતા પહેલા તેનું બોલવાનું પૂરું થાય ત્યાં સુધી શાંતિથી રાહ જુઓ.",
        },
        {
          "id": "SP20",
          "title": "સ્નેહપૂર્વક વિદાય",
          "desc":
              "અત્યારે જ તમારી સાથે નાની ચર્ચા કરેલી વ્યક્તિને હાથ હલાવીને 'બાય' કહો.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "મતોનો શોધક",
          "desc": "કોઈ પુસ્તક, સિનેમા કે ગીત પર કોઈનો અભિપ્રાય પૂછો.",
        },
        {
          "id": "L2",
          "title": "તપસીલ પૂછવી",
          "desc":
              "જો કોઈએ પોતાના વિશે કંઈક કહ્યું હોય, તો તેને જોડીને વધુ એક પ્રશ્ન પૂછો.",
        },
        {
          "id": "L3",
          "title": "ભલામણ",
          "desc":
              "નજીકમાં જમવા માટે સારું હોટેલ કયું છે તે જણાવવા કોઈ અજાણી વ્યક્તિને પૂછો.",
        },
        {
          "id": "L4",
          "title": "સમાન રુચિ",
          "desc": "કોઈની સાથે એક સમાન રુચિ શોધી કાઢો અને તેના પર ૨ મિનિટ બોલો.",
        },
        {
          "id": "L5",
          "title": "મદદનો હાથ",
          "desc":
              "કોઈ નાના કામમાં (બેગ ઉઠાવવા જેવા) મદદ કરવા માટે તમારી જાતે આગળ વધો.",
        },
        {
          "id": "L6",
          "title": "સામાજિક અવલોકન",
          "desc":
              "તમારા બંનેની આસપાસ બનતી કોઈ ઘટનાના આધારે સંભાષણની શરૂઆત કરો.",
        },
        {
          "id": "L7",
          "title": "ઓપન ક્વેશ્ચન",
          "desc":
              "કોઈને 'તમે આ વ્યવસાયમાં કે કામમાં કેવી રીતે આવ્યા?' તેમ પૂછો.",
        },
        {
          "id": "L8",
          "title": "સક્રિય સાંભળનાર",
          "desc":
              "કોઈનું બોલવું વચ્ચે અટકાવ્યા વગર ૩ મિનિટ સાંભળી લો, અને પછી તેણે શું કહ્યું તેનો ટૂંકો સાર કહો.",
        },
        {
          "id": "L9",
          "title": "એકત્રે હસવું",
          "desc": "એક નાના ગ્રુપને એક ટૂંકી, ગમ્મતભરી વાર્તા અથવા જોક કહો.",
        },
        {
          "id": "L10",
          "title": "જિજ્ઞાસા",
          "desc":
              "કોઈને તેઓ કયા ગામના છે અને તે ગામમાં તેમને શું ગમે છે તે પૂછો.",
        },
        {
          "id": "L11",
          "title": "સાચો રસ લેવો",
          "desc": "સહકર્મીને કામ સિવાયના તેના શોખ વિશે પૂછો.",
        },
        {
          "id": "L12",
          "title": "સહજ સલાહ",
          "desc":
              "જે વસ્તુમાં તમે નિપુણ છો તેવા એક વિષય પર કોઈને ઉપયોગી સલાહ આપો.",
        },
        {
          "id": "L13",
          "title": "ગ્રુપમાં સહમતી",
          "desc":
              "એક નાના ગ્રુપ ચર્ચામાં કોઈના મનને ટેકો આપવા માટે માથું હલાવો.",
        },
        {
          "id": "L14",
          "title": "સહજ આમંત્રણ",
          "desc": "કોઈને પૂછો કે 'બપોરના ભોજન માટે અમારી સાથે આવવું ગમશે?'",
        },
        {
          "id": "L15",
          "title": "પ્રામાણિક ભાવના વ્યક્ત કરવી",
          "desc":
              "કોઈને 'તમે જ્યારે X કર્યું ત્યારે મને ખૂબ જ આનંદ થયો હતો' તેમ કહીને તેનું કારણ સ્પષ્ટ કરો.",
        },
        {
          "id": "L16",
          "title": "જિજ્ઞાસાની જગ્યા",
          "desc":
              "કોઈને પૂછો કે 'હું હંમેશા વિચારું છું, એક્સ (X) ખરેખર કેવી રીતે કામ કરે છે?'",
        },
        {
          "id": "L17",
          "title": "નાના ગ્રુપ લીડ",
          "desc":
              "એક ગ્રુપમાં ૨ કે ૩ જણાને ઉત્તર આપવો પડે તેવો એક પ્રશ્ન પૂછો.",
        },
        {
          "id": "L18",
          "title": "ખરી પ્રશંસા",
          "desc":
              "કોઈના સ્વભાવ ગુણની પ્રશંસા કરો (દા.ત: 'તમે ખૂબ જ સારા સાંભળનાર છો').",
        },
        {
          "id": "L19",
          "title": "સમાન અનુભવ",
          "desc":
              "સંભાષણ દરમિયાન 'હું પણ એ જ પરિસ્થિતિમાં હતો' તેમ કહીને જોડાઓ.",
        },
        {
          "id": "L20",
          "title": "અર્થપૂર્ણ શાંતતા",
          "desc":
              "સંભાષણમાં શાંતિની ક્ષણ આવે તો તેને તરત જ શબ્દોથી ભરવાનો પ્રયત્ન કર્યા વગર સહજ રહેવા દો.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "હિંમતની શરૂઆત",
          "desc": "તમને બહુ ઓળખાણ ન હોય તેવી વ્યક્તિ સાથે સંભાષણની શરૂઆત કરો.",
        },
        {
          "id": "ST2",
          "title": "પ્રામાણિક સહભાગિતા",
          "desc": "એક ગ્રુપમાં એક ટૂંકી વ્યક્તિગત વાર્તા અથવા મત શેર કરો.",
        },
        {
          "id": "ST3",
          "title": "ચર્ચા કરવી",
          "desc":
              "કોઈના મત સાથે નમ્રતાપૂર્વક અસંમતિ દર્શાવો અને તેનું કારણ સ્પષ્ટ કરો.",
        },
        {
          "id": "ST4",
          "title": "ગ્રુપમાં સામેલ થવું",
          "desc":
              "ચાલુ રહેલા ગ્રુપ સંભાષણમાં સામેલ થાઓ અને એક વિચારપૂર્વકનું વાક્ય ઉમેરો.",
        },
        {
          "id": "ST5",
          "title": "ટોપિક શરૂ કરવો",
          "desc": "એક સામાજિક સમૂહમાં ચર્ચા માટે એક નવો વિષય માંડો.",
        },
        {
          "id": "ST6",
          "title": "સભામાં પ્રશ્ન પૂછવો",
          "desc": "એક જાહેર સભામાં અથવા ક્લાસરૂમના વાતાવરણમાં એક પ્રશ્ન પૂછો.",
        },
        {
          "id": "ST7",
          "title": "હિંમતની વિનંતી",
          "desc":
              "એક કેફે અથવા પાર્કમાં એક અજાણી વ્યક્તિને તેની બાજુમાં બેસી શકાય કે કેમ, તેમ પૂછો.",
        },
        {
          "id": "ST8",
          "title": "સંવાદનો પુલ",
          "desc":
              "એકબીજાને ઓળખતા ન હોય તેવા બે જણાની ઓળખાણ કરાવો અને તેમની વચ્ચે એક સમાન કડી શોધી કાઢો.",
        },
        {
          "id": "ST9",
          "title": "ખચકાટ વગર જરૂરિયાત વ્યક્ત કરવી",
          "desc":
              "તમને પરેશાન કરતી કોઈ બાબત રોકવા માટે અથવા કોઈને બાજુ પર ખસવા નમ્રતાપૂર્વક કહો.",
        },
        {
          "id": "ST10",
          "title": "વાર્તા કહેનાર",
          "desc":
              "૩ કે તેથી વધુ લોકો રહેલા ગ્રુપને એક વાર્તા કહેવામાં પુરોગામી બનો.",
        },
        {
          "id": "ST11",
          "title": "આહ્વાનાત્મક વિચાર રજૂ કરવો",
          "desc":
              "ગ્રુપમાં કોઈ સામાન્ય સમજને એક મૈત્રીપૂર્ણ અને આદરયુક્ત માર્ગે પડકાર આપો.",
        },
        {
          "id": "ST12",
          "title": "સામાજિક પુરોગામી",
          "desc":
              "એક ઓરડામાં પ્રવેશ કરતી વખતે બધાને પહેલા 'નમસ્તે' કહેનારા વ્યક્તિ તમે પોતે બનો.",
        },
        {
          "id": "ST13",
          "title": "સહાનુભૂતિથી સાંભળવું",
          "desc":
              "કોઈ પોતાના મનની નકારાત્મકતા કે ગુસ્સો વ્યક્ત કરતું હોય ત્યારે તે સાંભળો અને તેને સપોર્ટ આપતો જવાબ આપો.",
        },
        {
          "id": "ST14",
          "title": "જાહેર પ્રદર્શન",
          "desc":
              "એક સામાજિક ગેટ-ટુગેધરમાં તમારા મનગમતા વિષય પર ૧-૨ મિનિટ બોલો.",
        },
        {
          "id": "ST15",
          "title": "નબળાઈ સ્વીકારવી",
          "desc":
              "એક ગ્રુપ સામે તમે કોઈ બાબતમાં નર્વસ હતા તે સ્વીકારો અને તેના પર બધા મળીને હસો.",
        },
        {
          "id": "ST16",
          "title": "મર્યાદા નક્કી કરવી",
          "desc":
              "કોઈ મોટું સ્પષ્ટીકરણ આપ્યા વગર, તમારે જ્યાં જવું ન હોય તેવું આમંત્રણ નમ્રતાપૂર્વક નાકારો.",
        },
        {
          "id": "ST17",
          "title": "સક્રિય મધ્યસ્થ",
          "desc":
              "એક મતભેદમાં બે વ્યક્તિઓને એક મધ્યસ્થ નિર્ણય પર લાવવા માટે મદદ કરો.",
        },
        {
          "id": "ST18",
          "title": "જાહેર પ્રશંસા",
          "desc":
              "એક ગ્રુપમાં કોઈના પ્રયત્નોની અથવા મેળવેલી સફળતાની જાહેર પ્રશંસા કરો.",
        },
        {
          "id": "ST19",
          "title": "સીધો દ્રષ્ટિકોણ",
          "desc":
              "તમને જરૂરી કોઈ મદદ માટે અથવા સલાહ માટે કોઈને સીધી વિનંતી કરો.",
        },
        {
          "id": "ST20",
          "title": "સંભાષણને નવો વળાંક",
          "desc":
              "એક સંભાષણને બોરિંગ ટોપિક પરથી એક રસપ્રદ વિષય તરફ હળવેથી વાળો.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "ભેટ",
          "desc":
              "કોઈને એક નાની ગિફ્ટ અથવા ખાવાની વસ્તુ આપીને 'તમને આ ગમશે એવું વિચાર્યું હતું' તેમ કહો.",
        },
        {
          "id": "B2",
          "title": "હિંમતનું નેતૃત્વ",
          "desc":
              "એક નાના ગ્રુપના લોકોને એક પ્લાન અથવા કોઈ જગ્યા જોવાની કલ્પના સૂચવો.",
        },
        {
          "id": "B3",
          "title": "મૂલ્ય વ્યક્ત કરવું",
          "desc":
              "તમારા જીવનમાં તેમના હોવાનું મૂલ્ય તમે કેમ મહત્વનું માનો છો, તે કોઈને ખાસ રીતે કહો.",
        },
        {
          "id": "B4",
          "title": "સામાજિક સંયોજક",
          "desc":
              "કેટલાક લોકો માટે એક નાની મીટિંગ અથવા એક કોફી ડેટનું આયોજન કરો.",
        },
        {
          "id": "B5",
          "title": "ઊંડાણપૂર્વક સંવાદ",
          "desc":
              "કોઈની સાથે ૧五 મિનિટ કરતાં વધુ સમય માટે સખત અને અર્થપૂર્ણ સંવાદ સાધો.",
        },
        {
          "id": "B6",
          "title": "આત્મવિશ્વાસનું શિખર",
          "desc":
              "તમને જોતાં જ થોડો ડર કે સંકોચ થાય તેવી વ્યક્તિ સાથે સંભાષણની શરૂઆત કરો.",
        },
        {
          "id": "B7",
          "title": "જાહેર શુભેચ્છા",
          "desc":
              "એક ગ્રુપમાં એક ચોક્કસ વ્યક્તિ માટે ટૂંકું, સકારાત્મક પ્રશંસાનું વાક્ય કહો અથવા તેના પ્રયત્નોને આવકારો.",
        },
        {
          "id": "B8",
          "title": "મર્યાદા નક્કી કરનાર",
          "desc":
              "કોઈ પણ મોટા સ્પષ્ટીકરણ વગર, કોઈ અયોગ્ય માંગણીને દ્રઢતાથી પણ નમ્રતાથી 'ના' કહો.",
        },
        {
          "id": "B9",
          "title": "સીધી વિનંતી",
          "desc":
              "તમે માનતા કે આદર કરતા હોવ તેવા કોઈની સાથે ૧૦ મિનિટ બોલવા કે માર્ગદર્શન માટે વિનંતી કરો.",
        },
        {
          "id": "B10",
          "title": "ભાવનાત્મક પ્રયત્ન",
          "desc":
              "તમારા મિત્ર સાથે લાગણીઓ અથવા માનસિક સ્વાસ્થ્ય વિશે ઊંડી ચર્ચા શરૂ કરો.",
        },
        {
          "id": "B11",
          "title": "સામાજિક મધ્યસ્થ",
          "desc":
              "શાંત સંભાષણ દ્વારા બે વ્યક્તિઓ વચ્ચેનો એક નાનો વિવાદ મિટાવવા માટે મદદ કરો.",
        },
        {
          "id": "B12",
          "title": "ધૈર્યની પ્રશંસા",
          "desc":
              "તદ્દન અજાણી વ્યક્તિ સામે ઊભા રહીને તમે તેનામાં પ્રશંસનીય લાગતી એક બાબત કહો.",
        },
        {
          "id": "B13",
          "title": "નેટવર્કિંગ પગલું",
          "desc":
              "તમારા ક્ષેત્રના એક નિષ્ણાત અથવા યોગ્ય વ્યક્તિ સાથે પોતાની ઓળખાણ કરાવીને સલાહ માંગો.",
        },
        {
          "id": "B14",
          "title": "હિંમતનું સત્ય",
          "desc":
              "કોઈની સાથે કહેવા માટે કઠિન પણ સંબંધ માટે ફાયદાકારક હોય તેવું એક સાચું સત્ય બોલો.",
        },
        {
          "id": "B15",
          "title": "પૂર્ણ ખીલવું",
          "desc":
              "એક નાનો સામાજિક કાર્યક્રમ આયોજિત કરો અને ત્યાં આવનાર દરેક અતિથિ કમ્ફર્ટેબલ રહે તેની કાળજી લો.",
        },
        {
          "id": "B16",
          "title": "જાહેર વક્તા",
          "desc":
              "એક મીટિંગ અથવા ઇવેન્ટનો એક નાનો ભાગ લીડ કરવા કે બોલવા માટે તમારી જાતે આગળ આવો.",
        },
        {
          "id": "B17",
          "title": "અનુભવોના માર્ગદર્શક",
          "desc":
              "બીજાને પ્રોત્સાહિત કરવા માટે તમારી કઠિન પરિસ્થિતિનો અથવા તમે પાર કરેલી અસફળતાનો અનુભવ શેર કરો.",
        },
        {
          "id": "B18",
          "title": "ધૈર્યની માફી",
          "desc":
              "ભૂતકાળની એક ભૂલ માટે માફી માંગવા તમે જાતે સંભાષણની શરૂઆત કરો, ભલે તે ગમે તેટલું જૂનું કેમ ન હોય.",
        },
        {
          "id": "B19",
          "title": "માર્ગદર્શક (Mentor)",
          "desc":
              "તમારા કરતાં અનુભવમાં ઓછા હોય તેવા વ્યક્તિને કોઈ સ્કીલ કે કામમાં મદદ કરવા માટે આગળ રહો.",
        },
        {
          "id": "B20",
          "title": "સામાજિક શિલ્પકાર",
          "desc":
              "મિત્ર વર્તુળ માટે એક નવી સામાજિક પરંપરા અથવા વારંवार થતા એક ગેટ-ટુગેધરની (Meetup) શરૂઆત કરો.",
        },
      ],
    },
    'pa': {
      "Seedling": [
        {
          "id": "S1",
          "title": "ਪਹਿਲਾ ਕਦਮ",
          "desc":
              "ਅੱਜ ਕਿਸੇ ਇੱਕ ਵਿਅਕਤੀ ਨਾਲ ਅੱਖਾਂ ਮਿਲਾਓ ਅਤੇ ਉਸ ਵੱਲ ਦੇਖ ਕੇ ਮੁਸਕਰਾਓ।",
        },
        {
          "id": "S2",
          "title": "ਇੱਕ ਸਧਾਰਨ ਹੈਲੋ",
          "desc": "ਕਿਸੇ ਗੁਆਂਢੀ ਨੂੰ 'ਸ਼ੁਭ ਸਵੇਰ' ਜਾਂ 'ਸਤਿ ਸ਼੍ਰੀ ਅਕਾਲ' ਕਹੋ।",
        },
        {
          "id": "S3",
          "title": "ਧੰਨਵਾਦ",
          "desc": "ਕਿਸੇ ਦੁਕਾਨਦਾਰ ਨੂੰ ਸਾਫ਼ ਆਵਾਜ਼ ਵਿੱਚ 'ਧੰਨਵਾਦ' ਕਹੋ।",
        },
        {
          "id": "S4",
          "title": "ਨਿਰੀਖਣ",
          "desc":
              "ਕਿਸੇ ਅਣਜਾਣ ਵਿਅਕਤੀ ਵਿੱਚ ਕੋਈ ਸਕਾਰਾਤਮਕ ਗੱਲ ਨੋਟ ਕਰੋ ਅਤੇ ਮੁਸਕਰਾਓ।",
        },
        {
          "id": "S5",
          "title": "ਸ਼ਾਂਤੀ ਨਾਲ ਹੱਥ ਹਿਲਾਉਣਾ",
          "desc":
              "ਦੂਰੋਂ ਜਿਸ ਕਿਸੇ ਨੂੰ ਤੁਸੀਂ ਪਛਾਣਿਆ ਹੋਵੇ, ਉਸ ਵੱਲ ਹੌਲੀ ਜਿਹਾ ਹੱਥ ਹਿਲਾਓ।",
        },
        {
          "id": "S6",
          "title": "ਦਰਵਾਜ਼ਾ ਫੜ ਕੇ ਰੱਖਣਾ",
          "desc": "ਆਪਣੇ ਪਿੱਛੇ ਆਉਣ ਵਾਲੇ ਵਿਅਕਤੀ ਲਈ ਦਰਵਾਜ਼ਾ ਖੋਲ੍ਹ ਕੇ ਫੜੀ ਰੱਖੋ।",
        },
        {
          "id": "S7",
          "title": "ਸਿਰ ਹਿਲਾਉਣਾ",
          "desc":
              "ਕੋਲੋਂ ਲੰਘਦੇ ਸਮੇਂ ਕਿਸੇ ਸਹਿਕਰਮੀ ਨੂੰ ਪਿਆਰ ਨਾਲ ਸਿਰ ਹਿਲਾ ਕੇ ਨਮਸਕਾਰ ਕਰੋ।",
        },
        {
          "id": "S8",
          "title": "ਸ਼ੀਸ਼ੇ ਦਾ ਅਭਿਆਸ",
          "desc":
              "ਸ਼ੀਸ਼ੇ ਦੇ ਸਾਹਮਣੇ ਖੜ੍ਹੇ ਹੋ ਕੇ 1 ਮਿੰਟ ਲਈ ਆਪਣੀ 'ਆਤਮ-ਵਿਸ਼ਵਾਸ ਭਰੀ ਮੁਸਕਰਾਹਟ' ਦਾ ਅਭਿਆਸ ਕਰੋ।",
        },
        {
          "id": "S9",
          "title": "ਛੋਟੀ ਨਜ਼ਰ",
          "desc": "ਕਿਸੇ ਵੱਲ 2 ਸੈਕੰਡ ਦੇਖੋ, ਫਿਰ ਮੁਸਕਰਾ ਕੇ ਨਜ਼ਰ ਘੁਮਾ ਲਓ।",
        },
        {
          "id": "S10",
          "title": "ਸ਼ਾਂਤ ਪ੍ਰਸ਼ੰਸਾ",
          "desc": "ਕਿਸੇ ਦੀ ਸੋਸ਼ਲ ਮੀਡੀਆ ਪੋਸਟ ਦੇ ਹੇਠਾਂ ਇੱਕ ਵਧੀਆ ਕਮੈਂਟ ਲਿਖੋ।",
        },
        {
          "id": "S11",
          "title": "ਜਗ੍ਹਾ ਸਾਂਝੀ ਕਰਨੀ",
          "desc":
              "ਕਿਸੇ ਜਨਤਕ ਥਾਂ \'ਤੇ ਕਿਸੇ ਦੇ ਨਾਲ ਬੈਠੋ ਅਤੇ ਤੁਰੰਤ ਨਜ਼ਰ ਘੁਮਾਉਣ ਦੀ ਬਜਾਏ ਕੁਝ ਸਮਾਂ ਉੱਥੇ ਬਿਤਾਓ।",
        },
        {
          "id": "S12",
          "title": "ਸਧਾਰਨ ਨਿਮਰਤਾ",
          "desc":
              "ਗਲਿਆਰੇ ਵਿੱਚ ਕਿਸੇ ਦੇ ਕੋਲੋਂ ਲੰਘਦੇ ਸਮੇਂ ਨਿਮਰਤਾ ਨਾਲ 'ਐਕਸਕਿਊਜ਼ ਮੀ' ਕਹੋ।",
        },
        {
          "id": "S13",
          "title": "ਨਿੱਘਾ ਸੁਆਗਤ",
          "desc":
              "ਕਿਸੇ ਡਿਲੀਵਰੀ ਬੁਆਏ ਜਾਂ ਕੂਰੀਅਰ ਲਿਆਉਣ ਵਾਲੇ ਵਿਅਕਤੀ ਨੂੰ 'ਸਤਿ ਸ਼੍ਰੀ ਅਕਾਲ' ਕਹੋ।",
        },
        {
          "id": "S14",
          "title": "ਛੋਟਾ ਇਸ਼ਾਰਾ",
          "desc":
              "ਕਿਸੇ ਛੋਟੇ ਬੱਚੇ ਨੂੰ ਜਾਂ (ਮਾਲਕ ਦੀ ਇਜਾਜ਼ਤ ਨਾਲ) ਪਾਲਤੂ ਜਾਨਵਰ ਨੂੰ ਹੱਥ ਹਿਲਾ ਕੇ ਦਿਖਾਓ।",
        },
        {
          "id": "S15",
          "title": "ਨਰਮ ਮੁਸਕਰਾਹਟ",
          "desc": "ਅੱਜ ਤਿੰਨ ਵੱਖ-ਵੱਖ ਲੋਕਾਂ ਵੱਲ ਦੇਖ ਕੇ ਮੁਸਕਰਾਓ।",
        },
        {
          "id": "S16",
          "title": "ਆਈ ਕੰਟੈਕਟ ਚੈਲੇਂਜ",
          "desc":
              "ਕੈਸ਼ੀਅਰ ਦੇ ਪਹਿਲਾਂ ਨਜ਼ਰ ਘੁਮਾਉਣ ਤੱਕ ਉਸ ਨਾਲ ਅੱਖਾਂ ਮਿਲਾ ਕੇ ਰੱਖੋ।",
        },
        {
          "id": "S17",
          "title": "ਸ਼ਾਂਤ ਸਾਹ",
          "desc":
              "ਅੱਜ ਕਿਸੇ ਵੀ ਸਮਾਜਿਕ ਜਗ੍ਹਾ ਵਿੱਚ ਦਾਖਲ ਹੋਣ ਤੋਂ ਪਹਿਲਾਂ 3 ਵਾਰ ਡੂੰਘੇ ਸਾਹ ਲਓ।",
        },
        {
          "id": "S18",
          "title": "ਮੌਜੂਦਗੀ",
          "desc":
              "ਭੀੜ ਵਾਲੀ ਜਗ੍ਹਾ \'ਤੇ ਫੋਨ ਬਿਲਕੁਲ ਦੇਖੇ ਬਿਨਾਂ 5 ਮਿੰਟ ਲਈ ਸ਼ਾਂਤੀ ਨਾਲ ਖੜ੍ਹੇ ਰਹੋ।",
        },
        {
          "id": "S19",
          "title": "ਸਹਿਜਤਾ ਨਾਲ ਸਿਰ ਹਿਲਾਉਣਾ",
          "desc":
              "ਤੁਹਾਡੇ ਨਾਲ ਅੱਖਾਂ ਮਿਲਾਉਣ ਵਾਲੇ ਅਣਜਾਣ ਵਿਅਕਤੀ ਨੂੰ ਸਿਰ ਹਿਲਾ ਕੇ ਹੁੰਗਾਰਾ ਦਿਓ।",
        },
        {
          "id": "S20",
          "title": "ਵਧੀਆ ਵਿਦਾਈ",
          "desc":
              "ਦੁਕਾਨ ਵਿੱਚੋਂ ਬਾਹਰ ਨਿਕਲਦੇ ਸਮੇਂ ਕਿਸੇ ਨੂੰ 'ਤੁਹਾਡਾ ਦਿਨ ਚੰਗਾ ਲੰਘੇ' ਕਹੋ।",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "ਸੱਚੀ ਪ੍ਰਸ਼ੰਸਾ",
          "desc": "ਕਿਸੇ ਸਹਿਕਰਮੀ ਜਾਂ ਕਲਾਸਮੇਟ ਦੀ ਦਿਲੋਂ ਪ੍ਰਸ਼ੰਸਾ ਕਰੋ।",
        },
        {
          "id": "SP2",
          "title": "ਪ੍ਰਸ਼ਨ ਪੁੱਛਣਾ",
          "desc": "ਕਿਸе ਅਣਜਾਣ ਵਿਅਕਤੀ ਨੂੰ ਸਮਾਂ ਜਾਂ ਰਸਤਾ ਪੁੱਛੋ।",
        },
        {
          "id": "SP3",
          "title": "ਛੋਟੀ ਚਰਚਾ",
          "desc":
              "ਕਿਸੇ ਨੂੰ 'ਆਪਣਾ ਦਿਨ ਕਿਵੇਂ ਚੱਲ ਰਿਹਾ ਹੈ?' ਪੁੱਛੋ变更 ਅਤੇ ਉਸਦਾ ਜਵਾਬ ਧਿਆਨ ਨਾਲ ਸੁਣੋ।",
        },
        {
          "id": "SP4",
          "title": "ਮਦਦ ਮੰਗਣੀ",
          "desc": "ਕਿਸੇ ਖਾਸ ਚੀਜ਼ ਨੂੰ ਲੱਭਣ ਲਈ ਸਟੋਰ ਦੇ ਕਰਮਚਾਰੀ ਦੀ ਮਦਦ ਮੰਗੋ।",
        },
        {
          "id": "SP5",
          "title": "ਆਰਡਰ ਦਿੰਦੇ ਸਮੇਂ",
          "desc":
              "ਕੋਈ ਡ੍ਰਿੰਕ ਜਾਂ ਫੂਡ ਆਰਡਰ ਕਰੋ ਅਤੇ ਉੱਥੋਂ ਦੇ ਸਟਾਫ਼ ਨੂੰ ਪੁੱਛੋ ਕਿ ਉਹ ਕਿਵੇਂ ਹਨ।",
        },
        {
          "id": "SP6",
          "title": "ਪਛਾਣ ਕਰਵਾਉਣੀ",
          "desc": "ਆਪਣੇ ਇਲਾਕੇ ਦੇ ਕਿਸੇ ਨਵੇਂ ਵਿਅਕਤੀ ਨੂੰ ਆਪਣੀ ਪਛਾਣ ਕਰਵਾਓ।",
        },
        {
          "id": "SP7",
          "title": "ਮੌਸਮ ਬਾਰੇ ਗੱਲਾਂ",
          "desc":
              "ਲਾਈਨ ਵਿੱਚ ਇੰਤਜ਼ਾਰ ਕਰਦੇ ਸਮੇਂ ਨਾਲ ਵਾਲੇ ਵਿਅਕਤੀ ਨਾਲ ਮੌਸਮ ਬਾਰੇ ਗੱਲ ਕਰੋ।",
        },
        {
          "id": "SP8",
          "title": "ਸਧਾਰਨ ਪੁੱਛਗਿੱਛ",
          "desc": "ਸਹਿਕਰਮੀ ਨੂੰ ਪੁੱਛੋ ਕਿ 'ਤੁਸੀਂ ਵੀਕੈਂਡ \'ਤੇ ਕੀ ਕੀਤਾ?'",
        },
        {
          "id": "SP9",
          "title": "ਮਦਦ ਦੀ ਪੇਸ਼ਕਸ਼",
          "desc":
              "ਜੇਕਰ ਕੋਈ ਮੁਸ਼ਕਲ ਵਿੱਚ ਲੱਗੇ, ਤਾਂ ਉਸਨੂੰ ਪੁੱਛੋ 'ਕੀ ਤੁਹਾਨੂੰ ਇਸ ਵਿੱਚ ਕੋਈ ਮਦਦ ਚਾਹੀਦੀ ਹੈ?'",
        },
        {
          "id": "SP10",
          "title": "ਅਭਿਪ੍ਰਾਏ ਲੈਣਾ",
          "desc":
              "ਇੱਕ ਛੋਟੀ ਚੀਜ਼ ਦਿਖਾਉਂਦੇ ਹੋਏ ਦੋਸਤ ਨੂੰ ਪੁੱਛੋ 'ਤੁਹਾਨੂੰ ਇਸ ਬਾਰੇ ਕੀ ਲੱਗਦਾ ਹੈ?'",
        },
        {
          "id": "SP11",
          "title": "ਪੱਕਾ ਕਰਨਾ",
          "desc":
              "ਕਿਸੇ ਅਣਜਾਣ ਵਿਅਕਸ਼ਨ ਨਾਲ ਕੋਈ ਗੱਲ ਕਨਫਰਮ ਕਰੋ (ਜਿਵੇਂ: 'ਕੀ ਇਹ ਲਾਈਨ ਸਹੀ ਹੈ ਨਾ?')।",
        },
        {
          "id": "SP12",
          "title": "ਮਾਹੌਲ ਬਾਰੇ ਚਰਚਾ",
          "desc":
              "ਆਲੇ-ਦੁਆਲੇ ਦੇ ਮਾਹੌਲ ਬਾਰੇ ਇੱਕ ਛੋਟੀ ਟਿੱਪਣੀ ਕਰੋ (ਜਿਵੇਂ: 'ਇੱਥੇ ਸੱਚਮੁੱਚ ਬਹੁਤ ਭੀੜ ਹੈ')।",
        },
        {
          "id": "SP13",
          "title": "ਛੋਟੀ ਬੇਨਤੀ",
          "desc":
              "ਟੇਬਲ \'ਤੇ ਹੁੰਦੇ ਹੋਏ ਕਿਸੇ ਚੀਜ਼ (ਜਿਵੇਂ ਕਿ ਨੈਪਕਿਨ) ਨੂੰ ਆਪਣੀ ਤਰਫ਼ ਸਰਕਾਉਣ ਲਈ ਕਿਸੇ ਨੂੰ ਕਹੋ।",
        },
        {
          "id": "SP14",
          "title": "ਵਧੀਆ ਫੀਡਬੈਕ",
          "desc": "ਨਿਕਲਣ ਤੋਂ ਪਹਿਲਾਂ ਵੇਟਰ ਨੂੰ ਦੱਸੋ ਕਿ ਖਾਣਾ ਬਹੁਤ ਸ਼ਾਨਦਾਰ ਸੀ।",
        },
        {
          "id": "SP15",
          "title": "ਸਹਿਜ ਵਿਚਾਰ ਪੁੱਛਣਾ",
          "desc":
              "ਇੱਕ ਮਹੀਨੇ ਤੋਂ ਗੱਲ ਨਾ ਕੀਤੀ ਹੋਵੇ ਉਸ ਵਿਅਕਤੀ ਨੂੰ 'ਕਿਵੇਂ ਹੋ?' ਅਜਿਹਾ ਇੱਕ ਮੈਸੇਜ ਭੇਜੋ।",
        },
        {
          "id": "SP16",
          "title": "ਓਪਨ ਕੁਐਸਚਨ",
          "desc":
              "ਕਿਸੇ ਨੂੰ 'ਇਸ ਸ਼ਹਿਰ ਵਿੱਚ ਤੁਹਾਡੀ ਸਭ ਤੋਂ ਮਨਪਸੰਦ ਘੁੰਮਣ ਵਾਲੀ ਜਗ੍ਹਾ ਕਿਹੜੀ ਹੈ?' ਪੁੱਛੋ।",
        },
        {
          "id": "SP17",
          "title": "ਸਭ ਤੋਂ ਛੋਟਾ ਜੋਖਮ",
          "desc": "ਨੇੜੇ ਹੀ ਵਾਸ਼ਰੂਮ ਕਿੱਥੇ ਹੈ, ਇਹ ਕਿਸੇ ਅਣਜਾਣ ਵਿਅਕਤੀ ਨੂੰ ਪੁੱਛੋ।",
        },
        {
          "id": "SP18",
          "title": "ਚੀਜ਼ ਦੀ ਤਾਰੀਫ਼",
          "desc": "ਕਿਸੇ ਨੂੰ ਦੱਸੋ ਕਿ ਤੁਹਾਨੂੰ ਉਸਦੇ ਜੁੱਤੇ/ਬੈਗ/ਸਾਮਾਨ ਪਸੰਦ ਆਇਆ।",
        },
        {
          "id": "SP19",
          "title": "ਨਿਮਰਤਾ ਨਾਲ ਇੰਤਜ਼ਾਰ",
          "desc":
              "ਕਿਸੇ ਨੂੰ ਵੀ ਜਵਾਬ ਦੇਣ ਤੋਂ ਪਹਿਲਾਂ ਉਸਦਾ ਬੋਲਣਾ ਪੂਰਾ ਖ਼ਤਮ ਹੋਣ ਤੱਕ ਸ਼ਾਂਤੀ ਨਾਲ ਇੰਤਜ਼ਾਰ ਕਰੋ।",
        },
        {
          "id": "SP20",
          "title": "ਪਿਆਰ ਨਾਲ ਵਿਦਾਈ",
          "desc":
              "ਹੁਣੇ ਹੀ ਤੁਹਾਡੇ ਨਾਲ ਛੋਟੀ ਗੱਲਬਾਤ ਕਰਨ ਵਾਲੇ ਵਿਅਕਤੀ ਨੂੰ ਹੱਥ ਹਿਲਾ ਕੇ 'ਬਾਏ' ਕਹੋ।",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "ਵਿਚਾਰਾਂ ਦਾ ਖੋਜੀ",
          "desc": "ਇੱਕ ਕਿਤਾਬ, ਫ਼ਿਲਮ ਜਾਂ ਗੀਤ \'ਤੇ ਕਿਸੇ ਦਾ ਵਿਚਾਰ ਪੁੱਛੋ।",
        },
        {
          "id": "L2",
          "title": "ਵੇਰਵਾ ਪੁੱਛਣਾ",
          "desc":
              "ਜੇਕਰ ਕਿਸੇ ਨੇ ਆਪਣੇ ਬਾਰੇ ਕੁਝ ਦੱਸਿਆ ਹੋਵੇ, ਤਾਂ ਉਸਨੂੰ ਜੋੜ ਕੇ ਇੱਕ ਹੋਰ ਪ੍ਰਸ਼ਨ ਪੁੱਛੋ।",
        },
        {
          "id": "L3",
          "title": "ਸਿਫ਼ਾਰਸ਼",
          "desc":
              "ਨੇੜੇ ਹੀ ਖਾਣ ਲਈ ਕੋਈ ਵਧੀਆ ਹੋਟਲ ਕਿਹੜਾ ਹੈ, ਇਹ ਦੱਸਣ ਲਈ ਕਿਸੇ ਅਣਜਾਣ ਵਿਅਕਤੀ ਨੂੰ ਪੁੱਛੋ।",
        },
        {
          "id": "L4",
          "title": "ਸਾਂਝੀ ਪਸੰਦ",
          "desc": "ਕਿਸੇ ਨਾਲ ਇੱਕ ਸਾਂਝੀ ਦਿਲਚਸਪੀ ਲੱਭੋ ਅਤੇ ਉਸ \'ਤੇ 2 ਮਿੰਟ ਬੋਲੋ।",
        },
        {
          "id": "L5",
          "title": "ਮਦਦ ਦਾ ਹੱਥ",
          "desc":
              "ਇੱਕ ਛੋਟੇ ਕੰਮ ਵਿੱਚ (ਬੈਗ ਚੁੱਕਣ ਵਰਗੇ) ਮਦਦ ਕਰਨ ਲਈ ਆਪਣੇ ਆਪ ਅੱਗੇ ਵਧੋ।",
        },
        {
          "id": "L6",
          "title": "ਸਮਾਜਿਕ ਨਿਰੀਖਣ",
          "desc":
              "ਤੁਹਾਡੇ ਦੋਵਾਂ ਦੇ ਆਲੇ-ਦੁਆਲੇ ਵਾਪਰ ਰਹੀ ਕਿਸੇ ਘਟਨਾ ਦੇ ਆਧਾਰ \'ਤੇ ਗੱਲਬਾਤ ਦੀ ਸ਼ੁਰੂਆਤ ਕਰੋ।",
        },
        {
          "id": "L7",
          "title": "ਓਪਨ ਕੁਐਸਚਨ",
          "desc": "ਕਿਸੇ ਨੂੰ 'ਤੁਸੀਂ ਇਸ ਪੇਸ਼ੇ ਜਾਂ ਕੰਮ ਵਿੱਚ ਕਿਵੇਂ ਆਏ?' ਪੁੱਛੋ।",
        },
        {
          "id": "L8",
          "title": "ਸਰਗਰਮ ਸੁਣਨ ਵਾਲਾ",
          "desc":
              "ਕਿਸੇ ਦੀ ਗੱਲ ਵਿਚਕਾਰ ਰੋਕੇ ਬਿਨਾਂ 3 ਮਿੰਟ ਸੁਣ ਲਓ, ਅਤੇ ਬਾਅਦ ਵਿੱਚ ਉਸਨੇ ਕੀ ਦੱਸਿਆ ਉਸਦਾ ਸੰਖੇਪ ਸਾਰ ਕਹੋ।",
        },
        {
          "id": "L9",
          "title": "ਇਕੱਠੇ ਹੱਸਣਾ",
          "desc":
              "ਇੱਕ ਛੋਟੇ ਗਰੁੱਪ ਨੂੰ ਇੱਕ ਛੋਟੀ, ਮਜ਼ੇਦਾਰ ਕਹਾਣੀ ਜਾਂ ਚੁਟਕਲਾ ਸੁਣਾਓ।",
        },
        {
          "id": "L10",
          "title": "ਉਤਸੁਕਤਾ",
          "desc":
              "ਕਿਸੇ ਨੂੰ ਉਹ ਕਿਸ ਪਿੰਡ/ਸ਼ਹਿਰ ਦੇ ਹਨ ਅਤੇ ਉਸ ਜਗ੍ਹਾ ਵਿੱਚ ਉਨ੍ਹਾਂ ਨੂੰ ਕੀ ਪਸੰਦ ਹੈ, ਇਹ ਪੁੱਛੋ।",
        },
        {
          "id": "L11",
          "title": "ਸੱਚੀ ਦਿਲਚਸਪੀ",
          "desc": "ਸਹਿਕਰਮੀ ਨੂੰ ਕੰਮ ਤੋਂ ਇਲਾਵਾ ਉਸਦੇ ਸ਼ੌਕਾਂ ਬਾਰੇ ਪੁੱਛੋ।",
        },
        {
          "id": "L12",
          "title": "ਸਹਿਜ ਸਲਾਹ",
          "desc":
              "ਜਿਸ ਚੀਜ਼ ਵਿੱਚ ਤੁਸੀਂ ਮਾਹਰ ਹੋ, ਉਸ ਵਿਸ਼ੇ \'ਤੇ ਕਿਸੇ ਨੂੰ ਉਪਯੋਗੀ ਟਿਪ ਦਿਓ।",
        },
        {
          "id": "L13",
          "title": "ਗਰੁੱਪ ਵਿੱਚ ਸਹਿਮਤੀ",
          "desc":
              "ਇੱਕ ਛੋਟੇ ਗਰੁੱਪ ਦੀ ਚਰਚਾ ਵਿੱਚ ਕਿਸੇ ਦੇ ਵਿਚਾਰ ਦਾ ਸਮਰਥਨ ਕਰਨ ਲਈ ਸਿਰ ਹਿਲਾਓ।",
        },
        {
          "id": "L14",
          "title": "ਸਹਿਜ ਸੱਦਾ",
          "desc":
              "ਕਿਸੇ ਨੂੰ ਪੁੱਛੋ 'ਦੁਪਹਿਰ ਦੇ ਖਾਣੇ ਲਈ ਸਾਡੇ ਨਾਲ ਆਉਣਾ ਪਸੰਦ ਕਰੋਗੇ?'",
        },
        {
          "id": "L15",
          "title": "ਪ੍ਰਮਾਣਿਕ ਭਾਵਨਾ ਜ਼ਾਹਿਰ ਕਰਨੀ",
          "desc":
              "ਕਿਸੇ ਨੂੰ 'ਤੁਸੀਂ ਜਦੋਂ X ਕੀਤਾ ਸੀ ਤਾਂ ਮੈਨੂੰ ਬਹੁਤ ਖੁਸ਼ੀ ਹੋਈ ਸੀ' ਅਜਿਹਾ ਕਹਿ ਕੇ ਉਸਦਾ ਕਾਰਨ ਦੱਸੋ।",
        },
        {
          "id": "L16",
          "title": "ਉਤਸੁਕਤਾ ਦੀ ਜਗ੍ਹਾ",
          "desc":
              "ਕਿਸੇ ਨੂੰ ਪੁੱਛੋ 'ਮੈਂ ਹਮੇਸ਼ਾ ਸੋਚਦਾ ਹਾਂ, ਐਕਸ (X) ਅਸਲ ਵਿੱਚ ਕਿਵੇਂ ਕੰਮ ਕਰਦਾ ਹੈ?'",
        },
        {
          "id": "L17",
          "title": "ਛੋਟਾ ਗਰੁੱਪ ਲੀਡ",
          "desc":
              "ਇੱਕ ਗਰੁੱਪ ਵਿੱਚ 2 ਜਾਂ 3 ਜਣਿਆਂ ਨੂੰ ਉੱਤਰ ਦੇਣਾ ਪਵੇ ਅਜਿਹਾ ਇੱਕ ਪ੍ਰਸ਼ਨ ਪੁੱਛੋ।",
        },
        {
          "id": "L18",
          "title": "ਸੱਚੀ ਪ੍ਰਸ਼ੰਸਾ",
          "desc":
              "ਕਿਸੇ ਦੇ ਸੁਭਾਅ ਦੇ ਗੁਣ ਦੀ ਪ੍ਰਸ਼ੰਸਾ ਕਰੋ (ਜਿਵੇਂ: 'ਤੁਸੀਂ ਬਹੁਤ ਚੰਗੇ ਸੁਣਨ ਵਾਲੇ ਹੋ')।",
        },
        {
          "id": "L19",
          "title": "ਸਾਂਝਾ ਅਨੁਭਵ",
          "desc":
              "ਗੱਲਬਾਤ ਦੌਰਾਨ 'ਮੈਂ ਵੀ ਉਸ ਸਥਿਤੀ ਵਿੱਚ ਰਿਹਾ ਹਾਂ' ਅਜਿਹਾ ਕਹਿ ਕੇ ਜੁੜੋ।",
        },
        {
          "id": "L20",
          "title": "ਬਾਮਾਅਨਾ ਸ਼ਾਂਤੀ",
          "desc":
              "ਗੱਲਬਾਤ ਵਿੱਚ ਸ਼ਾਂਤੀ ਦਾ ਪਲ ਆਵੇ ਤਾਂ ਉਸਨੂੰ ਤੁਰੰਤ ਸ਼ਬਦਾਂ ਨਾਲ ਭਰਨ ਦੀ ਕੋਸ਼ਿਸ਼ ਕੀਤੇ ਬਿਨਾਂ ਸਹਿਜ ਰਹਿਣ ਦਿਓ।",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "ਹਿੰਮਤ ਦੀ ਸ਼ੁਰੂਆਤ",
          "desc":
              "ਤੁਹਾਨੂੰ ਜ਼ਿਆਦਾ ਜਾਣਕਾਰੀ ਨਾ ਹੋਵੇ ਉਸ ਵਿਅਕਤੀ ਨਾਲ ਗੱਲਬਾਤ ਦੀ ਸ਼ੁਰੂਆਤ ਕਰੋ।",
        },
        {
          "id": "ST2",
          "title": "ਪ੍ਰਮਾਣਿਕ ਸਹਿਭਾਗਤਾ",
          "desc": "ਇੱਕ ਗਰੁੱਪ ਵਿੱਚ ਇੱਕ ਛੋਟੀ ਨਿੱਜੀ ਕਹਾਣੀ ਜਾਂ ਮੱਤ ਸ਼ੇਅਰ ਕਰੋ।",
        },
        {
          "id": "ST3",
          "title": "ਚਰਚਾ ਕਰਨੀ",
          "desc":
              "ਕਿਸੇ ਦੇ ਵਿਚਾਰ ਨਾਲ ਨਿਮਰਤਾਪੂਰਵਕ ਅਸਹਿਮਤੀ ਜ਼ਾਹਿਰ ਕਰੋ变更 ਅਤੇ ਉਸਦਾ ਕਾਰਨ ਦੱਸੋ।",
        },
        {
          "id": "ST4",
          "title": "ਗਰੁੱਪ ਵਿੱਚ ਸ਼ਾਮਲ ਹੋਣਾ",
          "desc":
              "ਚੱਲ ਰਹੀ ਗਰੁੱਪ ਗੱਲਬਾਤ ਵਿੱਚ ਸ਼ਾਮਲ ਹੋਵੋ ਅਤੇ ਇੱਕ ਵਿਚਾਰਪੂਰਨ ਵਾਕ ਜੋੜੋ।",
        },
        {
          "id": "ST5",
          "title": "ਟੌਪਿਕ ਸ਼ੁਰੂ ਕਰਨਾ",
          "desc": "ਇੱਕ ਸਮਾਜਿਕ ਸਮੂਹ ਵਿੱਚ ਚਰਚਾ ਲਈ ਇੱਕ ਨਵਾਂ ਵਿਸ਼ਾ ਪੇਸ਼ ਕਰੋ।",
        },
        {
          "id": "ST6",
          "title": "ਸਭਾ ਵਿੱਚ ਪ੍ਰਸ਼ਨ ਪੁੱਛਣਾ",
          "desc":
              "ਇੱਕ ਜਨਤਕ ਸਭਾ ਵਿੱਚ ਜਾਂ ਕਲਾਸਰੂਮ ਦੇ ਮਾਹੌਲ ਵਿੱਚ ਇੱਕ ਪ੍ਰਸ਼ਨ ਪੁੱਛੋ।",
        },
        {
          "id": "ST7",
          "title": "ਹਿੰਮਤ ਦੀ ਬੇਨਤੀ",
          "desc":
              "ਇੱਕ ਕੈਫੇ ਜਾਂ ਪਾਰਕ ਵਿੱਚ ਇੱਕ ਅਣਜਾਣ ਵਿਅਕਤੀ ਨੂੰ ਪੁੱਛੋ ਕਿ ਉਸਦੇ ਸ਼ੇਜਾਰੀ ਬੈਠ ਸਕਦੇ ਹੋ।",
        },
        {
          "id": "ST8",
          "title": "ਸੰਵਾਦ ਦਾ ਪੁਲ",
          "desc":
              "ਇੱਕ ਦੂਜੇ ਨੂੰ ਨਾ ਜਾਣਦੇ ਹੋਣ ਵਾਲੇ ਦੋਹਾਂ ਦੀ ਪਛਾਣ ਕਰਵਾਓ ਅਤੇ ਉਨ੍ਹਾਂ ਵਿੱਚ ਇੱਕ ਸਾਂਝਾ ਧਾਗਾ ਲੱਭੋ।",
        },
        {
          "id": "ST9",
          "title": "ਠੋਸ ਲੋੜ ਜ਼ਾਹਿਰ ਕਰਨੀ",
          "desc":
              "ਤੁਹਾਨੂੰ ਪਰੇਸ਼ਾਨ ਕਰਨ ਵਾਲੀ ਚੀਜ਼ ਨੂੰ ਰੋਕਣ ਲਈ ਜਾਂ ਕਿਸੇ ਨੂੰ ਪਾਸੇ ਹੋਣ ਲਈ ਨਿਮਰਤਾ ਨਾਲ ਕਹੋ।",
        },
        {
          "id": "ST10",
          "title": "ਕਹਾਣੀਆਂ ਸੁਣਾਉਣ ਵਾਲਾ",
          "desc":
              "3 ਜਾਂ ਵਧੇਰੇ ਲੋਕਾਂ ਵਾਲੇ ਗਰੁੱਪ ਨੂੰ ਇੱਕ ਕਹਾਣੀ ਸੁਣਾਉਣ ਵਿੱਚ ਪਹਿਲ ਕਰੋ।",
        },
        {
          "id": "ST11",
          "title": "ਚੁਣੌਤੀਪੂਰਨ ਵਿਚਾਰ ਰੱਖਣਾ",
          "desc":
              "ਗਰੁੱਪ ਵਿੱਚ ਕਿਸੇ ਆਮ ਸਮਝ ਨੂੰ ਇੱਕ ਦੋਸਤਾਨਾ ਅਤੇ ਆਦਰਪੂਰਵਕ ਤਰੀਕੇ ਨਾਲ ਚੈਲੇਂج ਕਰੋ।",
        },
        {
          "id": "ST12",
          "title": "ਸਮਾਜਿਕ ਪਹਿਲਕਦਮੀ",
          "desc":
              "ਇੱਕ ਕਮਰੇ ਵਿੱਚ ਦਾਖਲ ਹੁੰਦੇ ਸਮੇਂ ਸਾਰਿਆਂ ਨੂੰ ਪਹਿਲਾਂ 'ਸਤਿ ਸ਼੍ਰੀ ਅਕਾਲ' ਕਹਿਣ ਵਾਲੇ ਵਿਅਕਤੀ ਤੁਸੀਂ ਖ਼ੁਦ ਬਣੋ।",
        },
        {
          "id": "ST13",
          "title": "ਹਮਦਰਦੀ ਨਾਲ ਸੁਣਨਾ",
          "desc":
              "ਕੋਈ ਆਪਣੇ ਮਨ ਦਾ ਦੁੱਖ ਜਾਂ ਗੁੱਸਾ ਦੱਸ ਰਿਹਾ ਹੋਵੇ ਤਾਂ ਉਹ ਸੁਣੋ ਅਤੇ ਉਸਨੂੰ ਸਹਾਰਾ ਦੇਣ ਵਾਲਾ ਜਵਾਬ ਦਿਓ।",
        },
        {
          "id": "ST14",
          "title": "ਜਨਤਕ ਪੇਸ਼ਕਾਰੀ",
          "desc":
              "ਇੱਕ ਸਮਾਜਿਕ ਗੈਟ-ਟੂਗੇਦਰ ਵਿੱਚ ਆਪਣੇ ਮਨਪਸੰਦ ਵਿਸ਼ੇ 'ਤੇ 1-2 ਮਿੰਟ ਬੋਲੋ।",
        },
        {
          "id": "ST15",
          "title": "ਕਮਜ਼ੋਰੀ ਮੰਨਣੀ",
          "desc":
              "ਇੱਕ ਗਰੁੱਪ ਸਾਹਮਣੇ ਤੁਸੀਂ ਕਿਸੇ ਚੀਜ਼ ਵਿੱਚ ਨਰਵਸ ਸੀ ਇਹ ਸਵੀਕਾਰ ਕਰੋ ਅਤੇ ਉਸ 'ਤੇ ਸਾਰੇ ਮਿਲ ਕੇ ਹੱਸੋ।",
        },
        {
          "id": "ST16",
          "title": "ਮਰਿਆਦਾ ਤੈਅ ਕਰਨੀ",
          "desc":
              "ਕੋਈ ਵੱਡਾ ਸਪਸ਼ਟੀਕਰਨ ਦਿੱਤੇ ਬਿਨਾਂ, ਜਿੱਥੇ ਤੁਸੀਂ ਨਾ ਜਾਣਾ ਚਾਹੁੰਦੇ ਹੋਵੋ ਉਹ ਸੱਦਾ ਨਿਮਰਤਾ ਨਾਲ ਨਕਾਰੋ।",
        },
        {
          "id": "ST17",
          "title": "ਸਰਗਰਮ ਮੱਧਸਥ",
          "desc":
              "ਇੱਕ ਮੱਤਭੇਦ ਵਿੱਚ ਦੋ ਵਿਅਕਤੀਆਂ ਨੂੰ ਇੱਕ ਸਾਂਝੇ ਫੈਸਲੇ 'ਤੇ ਲਿਆਉਣ ਲਈ ਮਦਦ ਕਰੋ।",
        },
        {
          "id": "ST18",
          "title": "ਜਨਤਕ ਸ਼ਲਾਘਾ",
          "desc":
              "ਇੱਕ ਗਰੁੱਪ ਵਿੱਚ ਕਿਸੇ ਦੇ ਯਤਨਾਂ ਦੀ ਜਾਂ ਪ੍ਰਾਪਤ ਕੀਤੀ ਸਫਲਤਾ ਦੀ ਜਨਤਕ ਤੌਰ 'ਤੇ ਪ੍ਰਸ਼ੰਸਾ ਕਰੋ।",
        },
        {
          "id": "ST19",
          "title": "ਸਿੱਧਾ ਦ੍ਰਿਸ਼ਟੀਕੋਣ",
          "desc":
              "ਤੁਹਾਨੂੰ ਲੋੜੀਂਦੀ ਮਦਦ ਲਈ ਜਾਂ ਸਲਾਹ ਲਈ ਕਿਸੇ ਨੂੰ ਸਿੱਧੀ ਬੇਨਤੀ ਕਰੋ।",
        },
        {
          "id": "ST20",
          "title": "ਗੱਲਬਾਤ ਨੂੰ ਨਵਾਂ ਮੋੜ",
          "desc":
              "ਇੱਕ ਗੱਲਬਾਤ ਨੂੰ ਬੋਰਿੰਗ ਟੌਪਿਕ ਤੋਂ ਇੱਕ ਦਿਲਚਸਪ ਵਿਸ਼ੇ ਵੱਲ ਹੌਲੀ ਜਿਹੀ ਮੋੜੋ।",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "ਤੋਹਫ਼ਾ",
          "desc":
              "ਕਿਸੇ ਨੂੰ ਇੱਕ ਛੋਟਾ ਗਿਫਟ ਜਾਂ ਖਾਣ ਵਾਲੀ ਚੀਜ਼ ਦੇ ਕੇ ਕਹੋ 'ਮੈਂ ਸੋਚਿਆ ਤੁਹਾਨੂੰ ਇਹ ਪਸੰਦ ਆਵੇਗਾ'।",
        },
        {
          "id": "B2",
          "title": "ਹਿੰਮਤ ਵਾਲਾ ਲੀਡਰਸ਼ਿਪ",
          "desc":
              "ਇੱਕ ਛੋਟੇ ਗਰੁੱਪ ਦੇ ਲੋਕਾਂ ਨੂੰ ਇੱਕ ਪਲਾਨ ਜਾਂ ਕੋਈ ਜਗ੍ਹਾ ਦੇਖਣ ਦਾ ਸੁਝਾਅ ਦਿਓ।",
        },
        {
          "id": "B3",
          "title": "ਮੁੱਲ ਜ਼ਾਹਿਰ ਕਰਨਾ",
          "desc":
              "ਆਪਣੀ ਜ਼ਿੰਦਗੀ ਵਿੱਚ ਉਨ੍ਹਾਂ ਦੇ ਹੋਣ ਦਾ ਮੁੱਲ ਤੁਸੀਂ ਕਿਉਂ ਮਹੱਤਵਪੂਰਨ ਮੰਨਦੇ ਹੋ, ਇਹ ਕਿਸੇ ਨੂੰ ਖਾਸ ਤਰੀਕੇ ਨਾਲ ਦੱਸੋ।",
        },
        {
          "id": "B4",
          "title": "ਸਮਾਜਿਕ ਪ੍ਰਬੰਧਕ",
          "desc": "ਕੁਝ ਲੋਕਾਂ ਲਈ ਇੱਕ ਛੋਟੀ ਮੀਟਿੰਗ ਜਾਂ ਇੱਕ ਕੌਫੀ ਡੇਟ ਦਾ ਆਯੋਜਨ ਕਰੋ।",
        },
        {
          "id": "B5",
          "title": "ਡੂੰਘੀ ਗੱਲਬਾਤ",
          "desc":
              "ਕਿਸੇ ਨਾਲ 15 ਮਿੰਟਾਂ ਤੋਂ ਵੱਧ ਸਮੇਂ ਲਈ ਡੂੰਘੀ ਅਤੇ ਅਰਥਪੂਰਨ ਗੱਲਬਾਤ ਕਰੋ।",
        },
        {
          "id": "B6",
          "title": "ਆਤਮ-ਵਿਸ਼ਵਾਸ ਦਾ ਸਿਖਰ",
          "desc":
              "ਤੁਹਾਨੂੰ ਦੇਖ ਕੇ ਚਿੰਤਾ ਜਾਂ ਸੰਕੋਚ ਹੋਵੇ ਅਜਿਹੇ ਵਿਅਕਤੀ ਨਾਲ ਗੱਲਬਾਤ ਦੀ ਸ਼ੁਰੂਆਤ ਕਰੋ।",
        },
        {
          "id": "B7",
          "title": "ਜਨਤਕ ਵਧਾਈ",
          "desc":
              "ਇੱਕ ਗਰੁੱਪ ਵਿੱਚ ਇੱਕ ਖਾਸ ਵਿਅਕਤੀ ਲਈ ਛੋਟਾ, ਸਕਾਰਾਤਮਕ ਤਾਰੀਫ਼ ਦਾ ਵਾਕ ਕਹੋ ਜਾਂ ਉਸਦੇ ਯਤਨਾਂ ਦਾ ਸੁਆਗਤ ਕਰੋ।",
        },
        {
          "id": "B8",
          "title": "ਮਰਿਆਦਾ ਰੱਖਣ ਵਾਲਾ",
          "desc":
              "ਕਿਸੇ ਵੀ ਵੱਡੇ ਸਪਸ਼ਟੀਕਰਨ ਤੋਂ ਬਿਨਾਂ, ਕਿਸੇ ਗਲਤ ਮੰਗ ਨੂੰ ਪੱਕੇ ਤੌਰ 'ਤੇ ਪਰ ਨਰਮੀ ਨਾਲ 'ਨਹੀਂ' ਕਹੋ।",
        },
        {
          "id": "B9",
          "title": "ਸਿੱਧੀ ਮੰਗ",
          "desc":
              "ਤੁਸੀਂ ਮੰਨਦੇ ਜਾਂ ਆਦਰ ਕਰਦੇ ਹੋ ਉਸ ਵਿਅਕਤੀ ਨਾਲ 10 ਮਿੰਟ ਬੋਲਣ ਜਾਂ ਮਾਰਗਦਰਸ਼ਨ ਲਈ ਬੇਨਤੀ ਕਰੋ।",
        },
        {
          "id": "B10",
          "title": "ਭਾਵਨਾਤਮਕ ਯਤਨ",
          "desc":
              "ਆਪਣੇ ਦੋਸਤ ਨਾਲ ਭਾਵਨਾਵਾਂ ਜਾਂ ਮਾਨਸਿਕ ਸਿਹਤ ਬਾਰੇ ਡੂੰਘੀ ਚਰਚਾ ਸ਼ੁਰੂ ਕਰੋ।",
        },
        {
          "id": "B11",
          "title": "ਸਮਾਜਿਕ ਵਿਚੋਲਾ",
          "desc":
              "ਸ਼ਾਂਤ ਗੱਲਬਾਤ ਰਾਹੀਂ ਦੋ ਵਿਅਕਤੀਆਂ ਵਿਚਕਾਰ ਇੱਕ ਛੋਟਾ ਵਿਵਾਦ ਸੁਲਝਾਉਣ ਵਿੱਚ ਮਦਦ ਕਰੋ।",
        },
        {
          "id": "B12",
          "title": "ਹਿੰਮਤ ਭਰੀ ਤਾਰੀਫ਼",
          "desc":
              "ਪੂਰੀ ਤਰ੍ਹਾਂ ਅਣਜਾਣ ਵਿਅਕਤੀ ਦੇ ਸਾਹਮਣੇ ਖੜ੍ਹੇ ਹੋ ਕੇ ਉਸ ਵਿੱਚ ਸਲਾਹੁਣਯੋਗ ਲੱਗਣ ਵਾਲੀ ਇੱਕ ਗੱਲ ਦੱਸੋ।",
        },
        {
          "id": "B13",
          "title": "ਨੈੱਟਵਰਕਿੰਗ ਕਦਮ",
          "desc":
              "ਆਪਣੇ ਖੇਤਰ ਦੇ ਇੱਕ ਮਾਹਰ ਜਾਂ ਯੋਗ ਵਿਅਕਤੀ ਨਾਲ ਆਪਣੀ ਪਛਾਣ ਕਰਵਾ ਕੇ ਸਲਾਹ ਮੰਗੋ।",
        },
        {
          "id": "B14",
          "title": "ਹਿੰਮਤ ਵਾਲਾ ਸੱਚ",
          "desc":
              "ਕਿਸੇ ਨੂੰ ਕਹਿਣ ਲਈ ਮੁਸ਼ਕਲ ਪਰ ਰਿਸ਼ਤੇ ਲਈ ਫਾਇਦੇਮੰਦ ਸੱਚੀ ਗੱਲ ਬੋਲੋ।",
        },
        {
          "id": "B15",
          "title": "ਪੂਰਾ ਖਿੜਨਾ",
          "desc":
              "ਇੱਕ ਛੋਟਾ ਸਮਾਜਿਕ ਪ੍ਰੋਗਰਾਮ ਆਯੋਜਿਤ ਕਰੋ ਅਤੇ ਉੱਥੇ ਆਉਣ ਵਾਲਾ ਹਰ ਮਹਿਮਾਨ ਕੰਫਰਟੇਬਲ ਰਹੇ ਇਸਦਾ ਧਿਆਨ ਰੱਖੋ।",
        },
        {
          "id": "B16",
          "title": "ਜਨਤਕ ਬੁਲਾਰਾ",
          "desc":
              "ਇੱਕ ਮੀਟਿੰਗ ਜਾਂ ਇਵੈਂਟ ਦਾ ਇੱਕ ਛੋਟਾ ਹਿੱਸਾ ਲੀਡ ਕਰਨ ਜਾਂ ਬੋਲਣ ਲਈ ਖ਼ੁਦ ਅੱਗੇ ਆਓ।",
        },
        {
          "id": "B17",
          "title": "ਅਨੁਭਵਾਂ ਦਾ ਗਾਈਡ",
          "desc":
              "ਦੂਜੇ ਨੂੰ ਉਤਸ਼ਾਹਿਤ ਕਰਨ ਲਈ ਆਪਣੀ ਮੁਸ਼ਕਲ ਸਥਿਤੀ ਜਾਂ ਪਾਰ ਕੀਤੀ ਹਾਰ ਦਾ ਅਨੁਭਵ ਸ਼ੇਅਰ ਕਰੋ।",
        },
        {
          "id": "B18",
          "title": "ਹਿੰਮਤ ਭਰੀ ਮਾਫ਼ੀ",
          "desc":
              "ਭੂਤਕਾਲ ਦੀ ਇੱਕ ਗਲਤੀ ਲਈ ਮਾਫ਼ੀ ਮੰਗਣ ਲਈ ਤੁਸੀਂ ਖ਼ੁਦ ਗੱਲਬਾਤ ਦੀ ਸ਼ੁਰੂਆਤ ਕਰੋ, ਚਾहे ਉਹ ਕਿੰਨੀ ਵੀ ਪੁਰਾਣੀ ਕਿਉਂ ਨਾ ਹੋਵੇ।",
        },
        {
          "id": "B19",
          "title": "ਮਾਰਗਦਰਸ਼ਕ",
          "desc":
              "ਆਪਣੇ ਨਾਲੋਂ ਅਨੁਭਵ ਵਿੱਚ ਘੱਟ ਵਿਅਕਤੀ ਨੂੰ ਕੋਈ ਸਕਿੱਲ ਜਾਂ ਕੰਮ ਵਿੱਚ ਮਦਦ ਕਰਨ ਲਈ ਅੱਗੇ ਰਹੋ।",
        },
        {
          "id": "B20",
          "title": "ਸਮਾਜਿਕ ਸ਼ਿਲਪਕਾਰ",
          "desc":
              "ਮਿੱਤਰ ਪਰਿਵਾਰ ਲਈ ਇੱਕ ਨਵੀਂ ਸਮਾਜਿਕ ਪਰੰਪਰਾ ਜਾਂ ਵਾਰ-ਵਾਰ ਹੋਣ ਵਾਲੇ ਗੈਟ-टुगेदर (Meetup) ਦੀ ਸ਼ੁਰੂਆਤ ਕਰੋ।",
        },
      ],
    },
    'mni': {
      "Seedling": [
        {
          "id": "S1",
          "title": "অহানবা খোঙথাঙ",
          "desc":
              "ঙসি মীওই অমগা মিৎ কুৎনা য়েংশিন্নগা মহাকপু য়েঙদুনা কোইনা নোক্রো।",
        },
        {
          "id": "S2",
          "title": "লাইবা হ্যালো অমা",
          "desc":
              "য়ুমলোন্নবা অমদা 'অয়ুক্কী খুরুমজরি' নত্ত্রগা 'খুরুમজরি' হায়য়ু।",
        },
        {
          "id": "S3",
          "title": "থাগৎপা ফোঙদোকপা",
          "desc": "দোকানদার অমদা ময়েক শেংনা 'থাগৎচরি' হায়না হায়য়ু।",
        },
        {
          "id": "S4",
          "title": "য়েংশিনবা",
          "desc": "খঙদবা মীওই অমগী অফবা মশক অমা খঙদোক্তুনা থমোইদগী নোক্রো।",
        },
        {
          "id": "S5",
          "title": "তুমিন্না খুনম খুৎ হিলাওবা",
          "desc":
              "লাপ্তগী নহাক্না খঙবা মীওই অমদা তপ্না খুৎ হিলাওদুনা থৌরাম তৌও।",
        },
        {
          "id": "S6",
          "title": "থোঙ ফাজিন্দুना থম্বা",
          "desc": "নಹಾ গী তুংদা লাক্লিবা মীওই অদুগীদমক থোঙ হাংদুना ফাজিনবীয়ু।",
        },
        {
          "id": "S7",
          "title": "মকোক নোনবা",
          "desc":
              "পাশতগী চৎপা মতমদা থবক্কী মরুপ অমদা নুংশিনা মकোক নোন্দুনা খুরুম্মু।",
        },
        {
          "id": "S8",
          "title": "অমিঙগী মমাঙদা প্র্যাকটিস",
          "desc":
              "অমিঙগী মমাঙদা লেপ্লগা ১ মিনিটকীদমক নહા গী 'ثাজজগী নোইবা' অসি প্র্যাকটিস তৌও।",
        },
        {
          "id": "S9",
          "title": "মচেৎ মিৎকুৎ",
          "desc": "কোনো অমদা ২ સેકન્ડ য়েঙৌ, মসি মতুং নোক্রগা মিৎ কুৎনা ওনখ্রৌ।",
        },
        {
          "id": "S10",
          "title": "তুমিন্না থাগৎপা",
          "desc":
              "কোনো অমগী সোশিয়েল মিডিয়া পোস্তকী মখাদা অফবা কমেন্ট অমা ইম্মু।",
        },
        {
          "id": "S11",
          "title": "মফম শেয়ার তৌবা",
          "desc":
              "মীয়াম মরক্কী মফমদা কোনো অমগী নাকলদা লেপ্লগা অমসুং তফনা মিৎ কুৎনা ওনখিগদবা নত্তে।",
        },
        {
          "id": "S12",
          "title": "লাইबा লৈবাকা চৎপা",
          "desc":
              "করিডোরদা কোনো অমগী নাকলদগী চৎপা মতমদা কনিৎনা 'মৈত্রেয়' হায়য়ু।",
        },
        {
          "id": "S13",
          "title": "নুংশিবা তরাম্না ওকপা",
          "desc":
              "ডিলિવરી বোয় নত্ত್ರগা কूरিয়র পুরকপা মীওই অদૂদা 'খুরুমজরি' হায়য়ু।",
        },
        {
          "id": "S14",
          "title": "অপীকপা সাইগা",
          "desc":
              "অঙাং অমা নত্ত್ರগা শেনবগী অয়াপাগા লোইননা শাফবা শা অমদা খুৎ હিলাওও।",
        },
        {
          "id": "S15",
          "title": "তপ্না নোইবা",
          "desc": "ঙসি তোঙান-তোঙানবা মীওই অহুমদা য়েঙদুনা নোক্রো।",
        },
        {
          "id": "S16",
          "title": "মিৎ কুৎনা য়েংশিনবগী চ্যালোঞ্জ",
          "desc":
              "ক্যাশিয়রনা অহানবা মিৎ কুৎনা ওনখ্রিবা ফাওবা মহাক্কা মিৎ কুৎনা য়েংশিন্নৌ।",
        },
        {
          "id": "S17",
          "title": "প্রশান্ত ইশ্বর শ্বাস",
          "desc":
              "ঙসি মীয়াম মরক্কী মফমদা চঙদ্রিঙဲ মমাঙদা ৩ বার চাউনা শ্বাস লৌও।",
        },
        {
          "id": "S18",
          "title": "মফমদা লৈবা",
          "desc":
              "মী য়াম্না তিনবা মফমদা ফোন অসি অমা খক্তસુ য়েঙদនា ৫ মিনিট লেপতুনা লৈয়ু।",
        },
        {
          "id": "S19",
          "title": "প্রাকৃতিক মকোক নোনবা",
          "desc":
              "নহাক্কা মিৎ কুৎਨਾ য়েংশিনখিবা খঙদባ মীওই অদૂদা মকোক নোন্দুনা খুরুম্মু।",
        },
        {
          "id": "S20",
          "title": "অফবা লোইশিনবা",
          "desc":
              "দোকানদগী থোক্লকপা মতমদা কোনো অমদা 'নਹਾ গী নুমিত অফবা ওইরসনু' হায়য়ু।",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "থემোইদগী থাগৎপা",
          "desc":
              "থবক্কী মরুপ অমা নত্ত્રগা ক্লাসমেত অমগী থমোইদগী অফবা থাগৎপা ফোঙদোক্কু।",
        },
        {
          "id": "SP2",
          "title": "ৱাহং হংবা",
          "desc": "খঙদবা মীওই অমদা মতম নত্ত্রগা লম্বী অসি হংমু।",
        },
        {
          "id": "SP3",
          "title": "অপীকপা খন্নবা",
          "desc":
              "কোনো অমদা 'নਹਾ গী নুমিত করম্না চৎলি?' হংমু অমসুং মহাক্কী উত্তর অদু থমোই কুৎনা লৈয়ু।",
        },
        {
          "id": "SP4",
          "title": "মচেৎ মথৌ তাবা",
          "desc":
              "দোকানগী থবক তৌবা মীওই অমদা অখন্নবা পোৎলम অমা থিবদা সপোর্ত তৌনবা হংমু।",
        },
        {
          "id": "SP5",
          "title": "অর্ডার পীবা মতমদা",
          "desc":
              "থক্নবা নտրগা চাক অর্ডার পীরগা মফম অদুগী স্তাফ অদুদা মহাক করম্না লৈবগে হংমু।",
        },
        {
          "id": "SP6",
          "title": "শক খঙহনবা",
          "desc":
              "নਹਾ গী মফমদা লৈরিবা অনৌবা মীওই অমা খক্তদা নহা গী মশাগী শক খঙহনৌ।",
        },
        {
          "id": "SP7",
          "title": "নুংশিৎ-মনীংগী ৱাতা",
          "desc":
              "লাইনদা কুইਨਾ ঙাইরিবা মতমদা নাকলদা লৈরিবা মীওই অদૂগা নুংশিৎ-মনীংগী মታংদা ৱারী শানৌ।",
        },
        {
          "id": "SP8",
          "title": "লাইবা থিবগী ৱারী",
          "desc": "থবक्कী মরুপ অদুদা 'উইকেন্ডদা করী তৌখিবગે?' হংমু।",
        },
        {
          "id": "SP9",
          "title": "মদদ পীবগী থৌরাং",
          "desc":
              "করিগুম্বা মীওই অমা অৱাবা লৈরমগুম খঙলগদি মহাকতা 'ঐহাক নহাকপু মদদ তৌগে?' হংমু।",
        },
        {
          "id": "SP10",
          "title": "মচেৎ লৌবা",
          "desc":
              "অপীকপা পোৎ অম উৎলগা মরুপ অদুদা 'নஹাক মসিগী মતાংদা করী খল্লিবগে?' হংমু।",
        },
        {
          "id": "SP11",
          "title": "অশেংবা খঙদোকপা",
          "desc":
              "খঙদবা মীওই অমগা ৱাফম অমা কনਫਰ্ম তৌও (દા.ત: 'মসি খক লাইন অচুম্ব্রা?')।",
        },
        {
          "id": "SP12",
          "title": "মফম অসিগী মતાংদা",
          "desc":
              "আলে-দুনীয়াগী মফম অসিগী মታংদা অপীকপা কমেন্ট অমা তৌও (દા.ત: 'মফম অসিদা অশেংবদা য়াম্না মী তিনখ্রে')।",
        },
        {
          "id": "SP13",
          "title": "অপীকপা সায় চাদবা",
          "desc":
              "টেবলদা লৈরিবা মতমদা পোৎ অমা (নেপকিনগুম্বা) নਹਾ গী নাকলদা থানবা কোনো অমদা হংমু।",
        },
        {
          "id": "SP14",
          "title": "অফबा ফীডবেক",
          "desc":
              "থোক্লকপা মমাঙদা ৱেটার অদুদা চাক অসি য়াম্না অফবা ওইখি হায়না হায়য়ু।",
        },
        {
          "id": "SP15",
          "title": "সহজ ১৫ নুমিতকী হংবা",
          "desc":
              "অমা থা খক ৱারী শানখਿদবা মীওই অদুদা 'করম্না লৈবগে?' হায়না মেসেজ অমা পীবীয়ু।",
        },
        {
          "id": "SP16",
          "title": "ওপন কুਐਸਚਨ",
          "desc":
              "কোনো অমদা 'শহর অসিદા নহা গী ખ্বাইদগী পামজবা কোইবা মফম কদাইনো?' হংমু।",
        },
        {
          "id": "SP17",
          "title": "খ্বাইদগী অপীকপা রিস্ক",
          "desc":
              "নাকলদা ৱাশরুম কদাইদা লৈবগে খঙব্রா হায়না খঙদবা মীওই অদুদা হংমু।",
        },
        {
          "id": "SP18",
          "title": "পোৎকী থাগৎপা",
          "desc":
              "কোনो অমদা মহাক্কী খুঙ্গাও/বেগ/এক্সেসরিজ অফবা ওইরে হায়না হায়য়ু।",
        },
        {
          "id": "SP19",
          "title": "মర్యాদা লৈনা ঙাইবা",
          "desc":
              "কোনো অমদা উত্তর পীবগী മমাঙদা মহাক্কী ৱারী পুম লোইশিনবা ফাওबा থমোই কুৎਨਾ ঙাইয়ু।",
        },
        {
          "id": "SP20",
          "title": "মৈত্রীপূর্ণ নিরোপ",
          "desc":
              "হৌজਿਕ খক নহাক্কা মচেৎ ৱারী শানখিবা মীওই অদুদা খুৎ হিলাওদুना 'বাই' হায়য়ু।",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "মচেৎ থিবগী লম্বী",
          "desc":
              "লাইরিক অমা, সিনেমা নত্ত্রগা ইশৈ অমগী মતાংদা কোনো অমগী মচেৎ হংমু।",
        },
        {
          "id": "L2",
          "title": "তপসিল হংবা",
          "desc":
              "করিগুম্বা কোনো অমনা মশাগী মతాংদা করী খক হায়রক্লগা মহাকপু লোইনนา মখা তানা ৱাহং അমা হংমু।",
        },
        {
          "id": "L3",
          "title": "সজেসন",
          "desc":
              "নাকলদা চাক চাবগী অফবা হোটেল কদাইদা লৈবগে তাক্নবা খঙদবা মীওই অমদা হংমু।",
        },
        {
          "id": "L4",
          "title": "సమాన ଆવଡ",
          "desc":
              "কোনো অমগা অমত্তা ওইবা পামজবা খঙদোক্তুনা মসিগী মতাংদা ২ মিনিট ৱারী শানৌ।",
        },
        {
          "id": "L5",
          "title": "মদদকী খুৎ",
          "desc":
              "অপীকপা থবক অমদা (বেগ পুথোকপগুম্বা) মদদ তৌনባ নહા গী মশাগী খুত্থাংদা মাঙদা চৎਲু।",
        },
        {
          "id": "L6",
          "title": "সোশিয়েল য়েংশিনবা",
          "desc": "নහා গী নাকলদা থোক্লিবা থৌদোক অমগী খুত্থাংদা ৱারী শানবা হৌও।",
        },
        {
          "id": "L7",
          "title": "ওপন কুਐਸਚਨ",
          "desc":
              "কোনো অমদা 'নഹাক থবক অসিদা নত্ত্রগা মসিদা করম্না লাকখিবগে?' হংমু।",
        },
        {
          "id": "L8",
          "title": "সক্রিয় তাবা মীওই",
          "desc":
              "কোনো অমগী ৱারী অসি মচেৎ লৈতনबा ৩ মিনিট তফনা তারগা মতুংদা মহাক্না করী হায়খিবগে মসি মচেৎ ওইনা হায়য়ু।",
        },
        {
          "id": "L9",
          "title": "অমত্তা ওইਨਾ নোইবা",
          "desc":
              "অপীকপা গ্রুপ অমদা নুংঙাইবা ৱারী അমা নত্ত্রগা জোকার অমা হায়য়ু।",
        },
        {
          "id": "L10",
          "title": "জিল্লেসা",
          "desc":
              "কোনো অমদা মহাক কদাইগী লৈবগেনো অমসুং মফম অদুদা মহাকপু করী പামজবগেনো হংmu।",
        },
        {
          "id": "L11",
          "title": "অশেংবা পামজबा",
          "desc":
              "থবক্কী মরুপ অদুদা থবক্কী মপানদা লৈরিবা মহাক্কী হোবীশিংগী মతాংদা হংমু।",
        },
        {
          "id": "L12",
          "title": "তপ্না সলাহ",
          "desc":
              "নਹাক্না খঙবা থবক অমগী ମতাংদা কোনো অমদা কান্নবা টিপ অমা পীবীয়ু।",
        },
        {
          "id": "L13",
          "title": "গ্রুপ মকোক নোਨባ",
          "desc":
              "অপীকপা গ্রুপ খন্নবদা কোনো অমগী মচেৎপু সপোর্ত তৌনবা মকোক নোন্নৌ।",
        },
        {
          "id": "L14",
          "title": "সহজ আমন্ত্রন",
          "desc":
              "কোনো অমদা 'ನುংথিল চাক চাবগী ঐখোয়গা লোইননা লাকপা পাম্ব্রা?' হংমু।",
        },
        {
          "id": "L15",
          "title": "প্রামাণিক থমোইগী ৱাফম",
          "desc":
              "কোনো অমদা 'নಹাক্না মমাঙদা X তৌখিবা মতमদা ঐહাক য়াম্না হরাওখি' হায়য়ু অমসুং মসিগী মরম অদু তাকৌ।",
        },
        {
          "id": "L16",
          "title": "জিল্লেসাগী মফম",
          "desc":
              "কোনো অমদা 'ঐহাক খল্লম্মী, এক্স (X) অসি অশেংবদা করম্না থবক তৌবগেনো?' হংমু।",
        },
        {
          "id": "L17",
          "title": "অপীকপা গ্রুপ লীড",
          "desc":
              "গ্রুপ অমদা ২ নত্ত্রগা ৩ জণানা উত্তর পীবা মথৌ তাবা ৱاهং অমা হংমু।",
        },
        {
          "id": "L18",
          "title": "অশেংবা থাগৎপা",
          "desc":
              "কোনো অমগী ক্যারেক্টারগী মশকপু থাগৎলু (દા.ત: 'নહাক য়াম্না অফबा তাবা মীওইনি')।",
        },
        {
          "id": "L19",
          "title": "অমত্তা ওইবা এক্সপিরিয়েন্স",
          "desc": "ৱারী শানবা মতমদা 'ঐહাকসু মਫম অদুদা লৈরম্মী' হায়না হায়য়ু।",
        },
        {
          "id": "L20",
          "title": "বামেয়ানা শান্ততা",
          "desc":
              "ৱারী শানবদা তুমিন্না লৈবা মতম অদু লাক্লগদি মসি ৱাহৈনা থুনা থন্নবা হোৎনদនា সহজ ওইনা লৈয়ু।",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "ਥৌনা লৈবা হৌবা",
          "desc": "নহাক্না হেন্না খঙদবা মীওই অমগা ৱারী শানবা হৌও।",
        },
        {
          "id": "ST2",
          "title": "প্রামাণিক শরুক য়াবা",
          "desc": "গ্রুপ অমদা অপীকপা মশাগী ৱারী নত্ত্রগা মচেৎ শেয়ার তৌও।",
        },
        {
          "id": "ST3",
          "title": "খন্ন-নৈবা",
          "desc":
              "কোনো অমগী মচেৎকা মৈত্রেয় ওইনা অরো তৌও অমসুং মসিগী মরম তাকৌ।",
        },
        {
          "id": "ST4",
          "title": "গ্রুপতা চঙবা",
          "desc":
              "চৎলিবা গ্রুপ ৱারী শানবদা শরুক য়াও অমসুং খল্লবা ৱাহৈ অমা হাপ্পীয়ু।",
        },
        {
          "id": "ST5",
          "title": "টোপিক হৌবা",
          "desc": "সামাজিক গ্রুপ অমদা খন্ননবা অনৌবা মরম অমা হৌও।",
        },
        {
          "id": "ST6",
          "title": "মীয়াম মরক্তা ৱاهং হংবা",
          "desc": "মীয়াম মরক্কী মীফমদা নত্ত্রগা ক্লাসরুমদা ৱাহং অমা হংমু।",
        },
        {
          "id": "ST7",
          "title": "হিম্মতকী রিকোয়েস্ট",
          "desc":
              "ক্যাফে নত্ত্রগা পার্ক অমদা খঙদባ মীওই অমদা মহাক্কী নাকলদা ফমবা য়াব্রা হায়ના হংমু।",
        },
        {
          "id": "ST8",
          "title": "যোগাযোগকী থোঙ",
          "desc":
              "অমত্তা খঙনদবা মীওই অনিবু শক খঙহনৌ অমসুং মখোয় অনিগী মরক্তা অমত্তা ওইবা মরম অমা থিবীয়u।",
        },
        {
          "id": "ST9",
          "title": "অশেংবা মথৌ তাবা",
          "desc":
              "নહাকপু অপনবা পীব পোৎলম অদু থিংনባ নত্ত്രগা কোনো অমদা নাকলদা চৎনবা মৈত্রেয় ওইনা হায়য়ু।",
        },
        {
          "id": "ST10",
          "title": "ৱারী হায়বা মীওই",
          "desc":
              "৩ নত্ত্রগা মসিদগী হেনবা মী লৈরিবা গ্রুপ অদূদা ৱারী অমা হায়বদা লীড লৌও।",
        },
        {
          "id": "ST11",
          "title": "আহ্বানাত্মেক ৱাফম",
          "desc":
              "গ্রুপ লৈরিবা মীয়াম মরক্কী মচেৎ অদুদা মরুপ-মপাংগুম অমসুং মৈত্রেয় ওইবা লম্বীনা চ্যালোঞ্জ তৌও।",
        },
        {
          "id": "ST12",
          "title": "সামাজিক পুদোৎপা",
          "desc":
              "কা অমদা চঙলকਪা মতমদা ময়াম মরক্তা খ্বাইদগী অহানબા 'খুরুमজরি' হায়বা মীওই অদু নഹাক মশামক ওইয়ু।",
        },
        {
          "id": "ST13",
          "title": "সহানুভূতিগা লোইননা তাবা",
          "desc":
              "কোনো অমনা মহাক্কী থমোইগী অৱাবা নত্ত્રগা সা ফোঙদোক্লকপা মতਮদা মসি তারগা সপোর্ত পীব মখলগী উত্তর পীবীয়ু।",
        },
        {
          "id": "ST14",
          "title": "মীয়াম মরক্তা উৎপা",
          "desc":
              "সামাজিক গেট-টুগেদার অমদা নഹാ গী পামজবা মরম অমগী মతాংদা ১-২ মিনিট ৱারী শানৌ।",
        },
        {
          "id": "ST15",
          "title": "নার্ভাস ওইবা স্বীকার তৌबा",
          "desc":
              "গ্রুপ অমগী মমাঙদা নహাক করিগুম্বা মতাংদা নার্ভাস ওইরম্মী হায়না স্বীকার তৌও অমসুং মসিদা ময়াম লোইননা নোক্রো।",
        },
        {
          "id": "ST16",
          "title": "মরী তৌবা থম্বা",
          "desc":
              "অমা চাউনা মরম তাকদនា, চৎপা පামদবা আমন্ত্রন অদু মৈত্রেয় ওইনা নালাওও।",
        },
        {
          "id": "ST17",
          "title": "সক্রিয় মধ্যস্থতা",
          "desc": "মচেৎ ওনবা মতাংদা মীওই অনিবু ময়ায় ওইবা মফমদা পুরকනባ মদদ তৌও।",
        },
        {
          "id": "ST18",
          "title": "মীয়াম মরক্তা থাগৎপা",
          "desc":
              "গ্রুপ অমদা কোনো অমগী থৌরাং নত্ত್ರগা মায় পাকপগী মতাংদা মীয়াম মরক্তা থাগৎলু।",
        },
        {
          "id": "ST19",
          "title": "সরাসরি লম্বী",
          "desc":
              "নഹাক মথৌ তাবা মদদ অমা নত্ত્રগা সলাহ অমগীদমক কোনো অমদা সরাসরি রিকোয়েস্ট তৌও।",
        },
        {
          "id": "ST20",
          "title": "ৱারী ওনখৎপা",
          "desc":
              "ৱারী শানবা অসি বোরিং ওইরবা মরমদগী নুংঙাইবা মরম অমদা তফনা ওনখৎပীয়ু।",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "খুদোল",
          "desc":
              "কোনো অমদা অপীকপা গিফ্ট নত্ত્રগা চাবগী পোৎলম পীরগা 'ঐহাক খল্লম্মী নහাক মসি পামগনি' হায়য়ু।",
        },
        {
          "id": "B2",
          "title": "হিম্মতকী লীডরশিপ",
          "desc":
              "অপীকপা গ্রুপ মীওইশিংদা প্ল্যান অমা নত্ত્રগা মফম অমা কোইবগী মਤਾংদা সজেসন পীবীয়ু।",
        },
        {
          "id": "B3",
          "title": "থাগৎচরিবা মশক",
          "desc":
              "নહા গী পুন্সীদা মহাক লৈবগী কান্নবা নஹাক্না করিগী থিজবগেনো, মসি কোনো অমদা অখন্ননা হায়য়ু।",
        },
        {
          "id": "B4",
          "title": "সামাজিক শেম-শাবা মীওই",
          "desc":
              "মীওই খরা খককীদমক অপীকপা মীনুং নত্ত્રগা কফি ডেট অমগী থৌরাং তৌও।",
        },
        {
          "id": "B5",
          "title": "খুল্লবা ৱারী শানবা",
          "desc":
              "কোনো অমগা অমত্তা ওইনা ১৫ মিনিটদগী হেন্না খুল্লবা অমসুং অর্থপূর্ণ ওইባ যোগাযোগ তৌও।",
        },
        {
          "id": "B6",
          "title": "থাজজগী চেরা",
          "desc":
              "নഹাকপু য়েঙলগা অকিबा নত্ত്രগা খরা লাকপা ফাওগদবা মীওই অমগা ৱারী শানবা হৌও।",
        },
        {
          "id": "B7",
          "title": "মীয়াম মরক্তা থাগৎপা",
          "desc":
              "গ্রুপ মীফমদা অখন্নবা মীওই অমগীদমক অপীকপা, পোজিতিব থাগৎপগী ৱাহৈ হায়য়ু নত্ত্রগা মহাক্কী থৌরাং ওকৌ।",
        },
        {
          "id": "B8",
          "title": "মরী ফাজিনবা মীওই",
          "desc":
              "অমা ചাউনা মরম তাকদនា, রিকোয়েস্ট অমদা কন অমসুং মৈত্রেয় ওইనా 'নালাও' হায়য়ু।",
        },
        {
          "id": "B9",
          "title": "সরাসরি রিকোয়েস্ট",
          "desc":
              "نહাক পামজबा নত্ত್ರগা ইকাইখুম্নবা মীওই অমগা ১০ মিনিট ৱারী শানবা নত্ত્રগা লম্বী তাক্নባ রিকোয়েস্ট তৌও।",
        },
        {
          "id": "B10",
          "title": "থমোইগী থৌরাং",
          "desc":
              "নহা গী মরুপ অমগা থমোইগী ৱাফম নত্ত্রগা মন মানসিক হকশেলগী মতাংদা খুল্লবা খන්නባ হৌও处理।",
        },
        {
          "id": "B11",
          "title": "সামাজিক মধ্যস্থতাকারী",
          "desc":
              "তপ্না ৱারী শানবগী খুত্থাংদা মীওই অনিগী মরক্তা লৈবা অপীকপা খৎনবা অসি কোকহনবদা মদদ তৌও।",
        },
        {
          "id": "B12",
          "title": "হিম্মতকী থাগৎপা",
          "desc":
              "পুম খঙনদባ মীওই অমগী মমাঙদা লেপ্লগা নહাক্না মহাকপু থমোইদগী থাজজবা মশক অমা হায়য়ু।",
        },
        {
          "id": "B13",
          "title": "নেটওয়ার্কিং খোঙথাঙ",
          "desc":
              "ਨહા গী থবক্কী মফমদা লৈবা মায় পাকপা নত্ত্রগা থাজবা য়াবা মীওই অমগা নહા গী শক খঙহনৌ অমসুং সलाহ হংমু।",
        },
        {
          "id": "B14",
          "title": "হিম্মতকী অশেংবা",
          "desc":
              "কোনো অমগা হায়বা অরুবা ওইরጋসু মরী অদুগীদমক কান্নগদবা অশেংবা ৱাফম অমা হায়য়ু।",
        },
        {
          "id": "B15",
          "title": "পূর্ণ সাতপা",
          "desc":
              "অপীকপা সামাজিক থৌরাম অমা শেম্মু অমসুং মফম অদুدا লাক্লিবা গেস্ট পুম্নমক কান্নবা ফাওনબા চেকশিন্নৌ।",
        },
        {
          "id": "B16",
          "title": "মীয়াম মরক্কী স্পীকর",
          "desc":
              "মীফম নত্ত್ರগা ইভেন্ট অমগী অপীকপা শরুক অমা লীড তৌনबा নত্ত്രগা ৱারী শাননবা নਹਾ মশামক মাঙদা লাকৌ।",
        },
        {
          "id": "B17",
          "title": "এক্সপিরিয়েন্সকী লম্বী তাকপা মীওই",
          "desc":
              "কোনো অমদা থৌনা পীনባ নਹਾ গী অৱাবা মতম নত্ত্রগা নഹাক্না মায় পাকখিবা হন্থখিবগী এক্সপিরিয়েন্স শেয়ার তৌও।",
        },
        {
          "id": "B18",
          "title": "হিম্মতকী মেকোইবা",
          "desc":
              "মমাঙগী অশোয়বা অমগীদমक মাফোই হংનबा নহা মশামক ৱারী শানবা হৌও, মসি অমা য়াম্না কুইখ্রবসু চৎনগনি।",
        },
        {
          "id": "B19",
          "title": "মেন্টর",
          "desc":
              "નહাকদগী এক্সপিরিয়েন্স হন্থবা মীওই অমদা স্কিল নত্ত્રগা থবক অমদা মদদ তৌনબા মাঙদা লৈয়ু।",
        },
        {
          "id": "B20",
          "title": "সামাজিক আর্কিটেক্ট",
          "desc":
              "মরুপ মপাংশিংগী গ্রুপকীদমক অনৌবা ফিক্স সামাজিক চৎনবী অমা নত্ত્રগা কুইਨਾ চৎকদবা অনৌবা মীতপ থৌরাম হৌও।",
        },
      ],
    },
    'my': {
      "Seedling": [
        {
          "id": "S1",
          "title": "Langkah Pertama",
          "desc": "Lakukan hubungan mata dan senyum kepada seorang hari ini.",
        },
        {
          "id": "S2",
          "title": "Sapaan Mudah",
          "desc": "Katakan 'Selamat pagi' atau 'Hello' kepada jiran.",
        },
        {
          "id": "S3",
          "title": "Ucapan Terima Kasih",
          "desc": "Katakan 'Terima kasih' dengan jelas kepada penjaga kedai.",
        },
        {
          "id": "S4",
          "title": "Pemerhatian",
          "desc":
              "Perhatikan sesuatu yang positif tentang orang asing dan senyum.",
        },
        {
          "id": "S5",
          "title": "Lambaian Senyap",
          "desc": "Lambai kepada seseorang yang anda kenali dari jauh.",
        },
        {
          "id": "S6",
          "title": "Pegang Pintu",
          "desc":
              "Pegang pintu supaya terbuka untuk seseorang di belakang anda.",
        },
        {
          "id": "S7",
          "title": "Anggukan",
          "desc":
              "Berikan anggukan mesra kepada rakan sekerja semasa anda berselisih dengan mereka.",
        },
        {
          "id": "S8",
          "title": "Cermin",
          "desc":
              "Latih 'senyuman yakin' anda di hadapan cermin selama 1 minit.",
        },
        {
          "id": "S9",
          "title": "Pandangan Sekilas",
          "desc":
              "Pandang seseorang selama 2 saat, kemudian senyum dan pandang ke arah lain.",
        },
        {
          "id": "S10",
          "title": "Pujian Senyap",
          "desc": "Tulis komen yang baik pada hantaran media sosial seseorang.",
        },
        {
          "id": "S11",
          "title": "Kongsi Ruang",
          "desc":
              "Duduk di sebelah seseorang di kawasan awam tanpa memandang ke arah lain dengan serta-merta.",
        },
        {
          "id": "S12",
          "title": "Penghargaan Mudah",
          "desc":
              "Katakan 'Tumpang lalu' dengan sopan apabila berselisih dengan seseorang di lalu lalang.",
        },
        {
          "id": "S13",
          "title": "Sapaan Mesra",
          "desc": "Katakan 'Hai' kepada pemandu penghantaran atau kurier.",
        },
        {
          "id": "S14",
          "title": "Lambaian Kecil",
          "desc":
              "Lambai kepada kanak-kanak atau haiwan peliharaan (dengan izin pemilik).",
        },
        {
          "id": "S15",
          "title": "Senyuman Lembut",
          "desc": "Senyum kepada tiga orang yang berbeza hari ini.",
        },
        {
          "id": "S16",
          "title": "Cabaran Hubungan Mata",
          "desc":
              "Kekalkan hubungan mata dengan juruwang sehingga mereka memandang ke arah lain terlebih dahulu.",
        },
        {
          "id": "S17",
          "title": "Nafas Tenang",
          "desc":
              "Tarik 3 nafas dalam-dalam sebelum memasuki ruang sosial hari ini.",
        },
        {
          "id": "S18",
          "title": "Kehadiran",
          "desc":
              "Berdiri di kawasan yang sesak selama 5 minit tanpa melihat telefon anda.",
        },
        {
          "id": "S19",
          "title": "Anggukan Kasual",
          "desc":
              "Angguk kepada orang asing yang membuat hubungan mata dengan anda.",
        },
        {
          "id": "S20",
          "title": "Suara Lembut",
          "desc":
              "Katakan 'Semoga hari anda menyenangkan' kepada seseorang semasa anda meninggalkan kedai.",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "Pujian",
          "desc":
              "Berikan pujian yang ikhlas kepada rakan sekerja atau rakan sekelas.",
        },
        {
          "id": "SP2",
          "title": "Pertanyaan",
          "desc": "Tanya orang asing tentang masa atau arah jalan.",
        },
        {
          "id": "SP3",
          "title": "Bual Kosong",
          "desc":
              "Tanya seseorang 'Bagaimana hari anda?' dan dengar jawapannya.",
        },
        {
          "id": "SP4",
          "title": "Permintaan",
          "desc": "Minta bantuan pekerja kedai untuk mencari barang tertentu.",
        },
        {
          "id": "SP5",
          "title": "Pesanan",
          "desc":
              "Pesan minuman atau makanan dan tanya kakitangan bagaimana keadaan mereka.",
        },
        {
          "id": "SP6",
          "title": "Suai Kenal",
          "desc":
              "Perkenalkan diri anda kepada seseorang yang baru di kawasan anda.",
        },
        {
          "id": "SP7",
          "title": "Bicara Cuaca",
          "desc":
              "Sebut tentang cuaca kepada seseorang semasa menunggu dalam barisan.",
        },
        {
          "id": "SP8",
          "title": "Pertanyaan Mudah",
          "desc":
              "Tanya rakan sekerja 'Apa yang anda lakukan sepanjang hujung minggu?'",
        },
        {
          "id": "SP9",
          "title": "Tawaran Bantuan",
          "desc":
              "Tanya seseorang 'Adakah anda memerlukan bantuan?' jika mereka kelihatan kesusahan.",
        },
        {
          "id": "SP10",
          "title": "Pendapat",
          "desc":
              "Tanya rakan 'Apa pendapat anda tentang ini?' mengenai satu objek kecil.",
        },
        {
          "id": "SP11",
          "title": "Pengesahan",
          "desc":
              "Sahkan satu butiran dengan orang asing (cth., 'Adakah ini barisan yang betul?').",
        },
        {
          "id": "SP12",
          "title": "Ruang Dikongsi",
          "desc":
              "Buat ulasan kecil tentang persekitaran (cth., 'Tempat ini sangat sesak').",
        },
        {
          "id": "SP13",
          "title": "Pertolongan Kecil",
          "desc":
              "Minta seseorang menghulurkan sesuatu (seperti tisu) kepada anda di meja.",
        },
        {
          "id": "SP14",
          "title": "Maklum Balas Positif",
          "desc":
              "Beritahu pelayan bahawa makanan itu sangat sedap sebelum keluar.",
        },
        {
          "id": "SP15",
          "title": "Sapaan Kasual",
          "desc":
              "Hantar mesej 'Apa khabar?' kepada seseorang yang sudah sebulan tidak anda hubungi.",
        },
        {
          "id": "SP16",
          "title": "Soalan Terbuka",
          "desc":
              "Tanya seseorang 'Di manakah tempat kegemaran anda untuk dilawati di bandar ini?'",
        },
        {
          "id": "SP17",
          "title": "Risiko Terkecil",
          "desc": "Tanya orang asing jika mereka tahu di mana tandas terdekat.",
        },
        {
          "id": "SP18",
          "title": "Pujian Barang",
          "desc":
              "Beritahu seseorang bahawa anda menyukai kasut/beg/aksesori mereka.",
        },
        {
          "id": "SP19",
          "title": "Henti Sekejap",
          "desc":
              "Tunggu sehingga seseorang selesai bercakap sepenuhnya sebelum memberikan respons kepada mereka.",
        },
        {
          "id": "SP20",
          "title": "Lambaian Mesra",
          "desc":
              "Lambai dan katakan 'Bye' kepada seseorang yang baru sahaja berinteraksi seketika dengan anda.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "Pencari Pendapat",
          "desc": "Minta pendapat seseorang tentang buku, filem, atau lagu.",
        },
        {
          "id": "L2",
          "title": "Butiran Tambahan",
          "desc":
              "Tanya soalan susulan selepas seseorang menceritakan sesuatu tentang diri mereka kepada anda.",
        },
        {
          "id": "L3",
          "title": "Cadangan Tempat",
          "desc":
              "Minta cadangan daripada orang asing tentang tempat makan yang sedap berdekatan.",
        },
        {
          "id": "L4",
          "title": "Persamaan",
          "desc":
              "Cari minat yang sama dengan seseorang dan bincangkan perkara itu selama 2 minit.",
        },
        {
          "id": "L5",
          "title": "Bantuan Tangan",
          "desc":
              "Tawarkan diri untuk membantu seseorang dengan tugasan kecil (seperti mengangkat beg).",
        },
        {
          "id": "L6",
          "title": "Pemerhatian Sosial",
          "desc":
              "Mulakan perbualan berdasarkan sesuatu yang berlaku di sekeliling anda berdua.",
        },
        {
          "id": "L7",
          "title": "Soalan Terbuka Mendalam",
          "desc":
              "Tanya seseorang 'Bagaimana anda mula menceburi bidang kerja ini?'",
        },
        {
          "id": "L8",
          "title": "Pendengar Aktif",
          "desc":
              "Dengar cerita seseorang selama 3 minit tanpa mencelah, kemudian rumuskan apa yang mereka katakan.",
        },
        {
          "id": "L9",
          "title": "Ketawa Bersama",
          "desc":
              "Ceritakan satu cerita pendek yang lucu atau lawak jenaka kepada kumpulan kecil.",
        },
        {
          "id": "L10",
          "title": "Sifat Ingin Tahu",
          "desc":
              "Tanya seseorang dari mana mereka berasal dan apa yang mereka suka tentang tempat tersebut.",
        },
        {
          "id": "L11",
          "title": "Minat yang Ikhlas",
          "desc":
              "Tanya rakan sekerja tentang hobi mereka di luar waktu kerja.",
        },
        {
          "id": "L12",
          "title": "Nasihat Ringan",
          "desc":
              "Berikan tip yang berguna kepada seseorang tentang perkara yang anda mahir.",
        },
        {
          "id": "L13",
          "title": "Anggukan Kumpulan",
          "desc":
              "Setujui pandangan seseorang dalam perbincangan kumpulan kecil.",
        },
        {
          "id": "L14",
          "title": "Ajakan Kasual",
          "desc":
              "Tanya seseorang 'Adakah anda mahu menyertai kami untuk makan tengah hari?'",
        },
        {
          "id": "L15",
          "title": "Refleksi Jujur",
          "desc":
              "Beritahu seseorang 'Saya sangat menghargai apabila anda melakukan X' dan jelaskan sebabnya.",
        },
        {
          "id": "L16",
          "title": "Persoalan Menarik",
          "desc":
              "Tanya seseorang 'Saya selalu tertanya-tanya, bagaimana X sebenarnya berfungsi?'",
        },
        {
          "id": "L17",
          "title": "Peneraju Kumpulan Kecil",
          "desc":
              "Tanya soalan yang memerlukan 2 atau 3 orang dalam kumpulan untuk menjawabnya.",
        },
        {
          "id": "L18",
          "title": "Pujian Tulen",
          "desc":
              "Puji seseorang tentang sifat peribadi mereka (cth., 'Anda seorang pendengar yang hebat').",
        },
        {
          "id": "L19",
          "title": "Pengalaman Dikongsi",
          "desc":
              "Katakan 'Saya juga pernah berada dalam situasi itu' semasa berbual.",
        },
        {
          "id": "L20",
          "title": "Jeda Bermakna",
          "desc":
              "Biarkan kesunyian berlaku dalam perbualan tanpa tergesa-gesa untuk mengisinya.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "Permulaan Berani",
          "desc":
              "Mulakan perbualan dengan seseorang yang anda tidak kenali rapat.",
        },
        {
          "id": "ST2",
          "title": "Perkongsian Jujur",
          "desc":
              "Kongsi cerita peribadi atau pendapat kecil dalam suasana kumpulan.",
        },
        {
          "id": "ST3",
          "title": "Perbincangan",
          "desc":
              "Nyatakan ketidaksetujuan secara sopan terhadap pendapat seseorang dan jelaskan sebabnya.",
        },
        {
          "id": "ST4",
          "title": "Menyertai Kumpulan",
          "desc":
              "Sertai perbualan kumpulan dan berikan sumbangan satu ayat yang bernas.",
        },
        {
          "id": "ST5",
          "title": "Peneraju Topik",
          "desc": "Bawa topik perbualan baharu dalam kumpulan sosial.",
        },
        {
          "id": "ST6",
          "title": "Soalan Terbuka Awam",
          "desc":
              "Tanya soalan dalam mesyuarat awam atau dalam suasana bilik darjah.",
        },
        {
          "id": "ST7",
          "title": "Permintaan Berani",
          "desc":
              "Tanya orang asing jika anda boleh duduk di sebelah mereka di kafe atau taman.",
        },
        {
          "id": "ST8",
          "title": "Jambatan Perbualan",
          "desc":
              "Perkenalkan dua orang yang tidak mengenali antara satu sama lain dan cari satu persamaan.",
        },
        {
          "id": "ST9",
          "title": "Keperluan Tegas",
          "desc":
              "Minta seseorang secara sopan untuk beralih atau berhenti melakukan sesuatu yang mengganggu anda.",
        },
        {
          "id": "ST10",
          "title": "Pencerita",
          "desc":
              "Ambil peranan utama dalam menceritakan kisah kepada kumpulan yang terdiri daripada 3 orang atau lebih.",
        },
        {
          "id": "ST11",
          "title": "Cabaran Terbuka",
          "desc":
              "Cabar pendapat umum dalam kumpulan dengan cara yang mesra dan penuh hormat.",
        },
        {
          "id": "ST12",
          "title": "Inisiatif Sosial",
          "desc":
              "Menjadi orang pertama yang mengucapkan 'Hello' kepada semua orang semasa memasuki bilik.",
        },
        {
          "id": "ST13",
          "title": "Mendengar dengan Empati",
          "desc":
              "Dengar seseorang meluahkan perasaan mereka dan berikan respons yang menyokong.",
        },
        {
          "id": "ST14",
          "title": "Pembentangan Awam",
          "desc":
              "Bercakap selama 1-2 minit tentang topik yang anda gemari dalam perjumpaan sosial.",
        },
        {
          "id": "ST15",
          "title": "Perkongsian Telus",
          "desc":
              "Akui kepada kumpulan bahawa anda gementar tentang sesuatu perkara, dan ketawa bersama-sama.",
        },
        {
          "id": "ST16",
          "title": "Tetapkan Sempadan",
          "desc":
              "Tolak jemputan yang anda tidak mahu hadiri secara sopan tanpa memberi penjelasan yang berlebihan.",
        },
        {
          "id": "ST17",
          "title": "Perantara Aktif",
          "desc":
              "Bantu dua orang mencari jalan tengah dalam sesuatu perselisihan faham.",
        },
        {
          "id": "ST18",
          "title": "Pujian Terbuka Awam",
          "desc":
              "Puji usaha atau pencapaian seseorang secara terbuka dalam kumpulan.",
        },
        {
          "id": "ST19",
          "title": "Pendekatan Terus",
          "desc":
              "Minta bantuan atau nasihat yang anda perlukan secara terus daripada seseorang.",
        },
        {
          "id": "ST20",
          "title": "Pusingan Perbualan",
          "desc":
              "Ubah perbualan dengan lancar daripada topik yang membosankan kepada topik yang menarik.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "Hadiah Kecil",
          "desc":
              "Berikan sedikit buah tangan kepada seseorang dan katakan 'Saya rasa anda akan menyukainya'.",
        },
        {
          "id": "B2",
          "title": "Cadangan Berani",
          "desc":
              "Cadangkan satu rancangan atau tempat untuk dilawati kepada sekumpulan kecil orang.",
        },
        {
          "id": "B3",
          "title": "Penghargaan",
          "desc":
              "Beritahu seseorang secara khusus mengapa anda menghargai kehadiran mereka dalam hidup anda.",
        },
        {
          "id": "B4",
          "title": "Hos Sosial",
          "desc":
              "Anjurkan satu perjumpaan kecil atau temu janji minum kopi untuk beberapa orang.",
        },
        {
          "id": "B5",
          "title": "Sembang Mendalam",
          "desc":
              "Lakukan perbualan yang mendalam dan bermakna dengan seseorang selama lebih daripada 15 minit.",
        },
        {
          "id": "B6",
          "title": "Kemuncak Keyakinan",
          "desc":
              "Mulakan perbualan dengan seseorang yang anda rasakan agak digeruni.",
        },
        {
          "id": "B7",
          "title": "Ucapan Penghormatan",
          "desc":
              "Berikan ucapan penghormatan yang singkat dan positif atau pujian terbuka kepada seseorang dalam kumpulan.",
        },
        {
          "id": "B8",
          "title": "Penegas Sempadan",
          "desc":
              "Katakan 'Tidak' kepada satu permintaan dengan tegas tetapi lembut, tanpa memberi penjelasan berlebihan.",
        },
        {
          "id": "B9",
          "title": "Permintaan Terus",
          "desc":
              "Minta seseorang yang anda kagumi untuk berbual selama 10 minit atau meminta bimbingan.",
        },
        {
          "id": "B10",
          "title": "Peneraju Emosi",
          "desc":
              "Mulakan perbualan tentang perasaan atau kesihatan mental dengan rakan anda.",
        },
        {
          "id": "B11",
          "title": "Perantara Sosial",
          "desc":
              "Bantu dua orang menyelesaikan konflik kecil melalui perbualan yang tenang.",
        },
        {
          "id": "B12",
          "title": "Pujian Berani",
          "desc":
              "Beritahu orang asing tentang sesuatu perkara yang anda benar-benar kagumi tentang dirinya.",
        },
        {
          "id": "B13",
          "title": "Langkah Jaringan",
          "desc":
              "Perkenalkan diri anda kepada profesional dalam bidang anda dan minta nasihat.",
        },
        {
          "id": "B14",
          "title": "Kebenaran Berani",
          "desc":
              "Beritahu seseorang tentang perkara sebenar yang sukar tetapi membantu untuk kebaikan hubungan.",
        },
        {
          "id": "B15",
          "title": "Mekar Sepenuhnya",
          "desc":
              "Anjurkan satu acara sosial kecil dan pastikan setiap tetamu merasa dialu-alukan.",
        },
        {
          "id": "B16",
          "title": "Penceramah Awam",
          "desc":
              "Tawarkan diri secara sukarela untuk bercakap atau memimpin sebahagian kecil mesyuarat atau acara.",
        },
        {
          "id": "B17",
          "title": "Laluan Telus",
          "desc":
              "Kongsi keperitan yang telah anda harungi untuk memberi galakan kepada orang lain.",
        },
        {
          "id": "B18",
          "title": "Permohonan Maaf Berani",
          "desc":
              "Mulakan perbualan untuk meminta maaf atas kesilapan lalu, walaupun perkara itu sudah lama berlaku.",
        },
        {
          "id": "B19",
          "title": "Pembimbing (Mentor)",
          "desc":
              "Tawarkan diri untuk membantu seseorang yang kurang berpengalaman daripada anda dalam sesuatu kemahiran.",
        },
        {
          "id": "B20",
          "title": "Arkitek Sosial",
          "desc":
              "Cipta satu tradisi sosial baharu atau perjumpaan berulang untuk sekumpulan rakan.",
        },
      ],
    },
    'th': {
      "Seedling": [
        {
          "id": "S1",
          "title": "ก้าวแรก",
          "desc": "สบตาและส่งยิ้มให้ใครบางคนหนึ่งคนในวันนี้",
        },
        {
          "id": "S2",
          "title": "คำทักทายง่ายๆ",
          "desc":
              "กล่าวคำว่า 'สวัสดีครับ/ค่ะ' หรือ 'อรุณสวัสดิ์' กับเพื่อนบ้าน",
        },
        {
          "id": "S3",
          "title": "คำขอบคุณ",
          "desc": "กล่าวคำว่า 'ขอบคุณครับ/ค่ะ' อย่างชัดเจนกับพนักงานขายของ",
        },
        {
          "id": "S4",
          "title": "การสังเกต",
          "desc": "สังเกตสิ่งดีๆ เกี่ยวกับคนแปลกหน้าแล้วส่งยิ้มให้เขา",
        },
        {
          "id": "S5",
          "title": "โบกมือเงียบๆ",
          "desc": "โบกมือทักทายคนรู้จักที่คุณมองเห็นจากระยะไกล",
        },
        {
          "id": "S6",
          "title": "เปิดประตูค้างไว้",
          "desc": "เปิดประตูค้างไว้ให้คนที่เดินตามหลังคุณมา",
        },
        {
          "id": "S7",
          "title": "การพยักหน้า",
          "desc":
              "พยักหน้าทักทายอย่างเป็นมิตรให้เพื่อนร่วมงานในตอนที่เดินสวนกัน",
        },
        {
          "id": "S8",
          "title": "หน้ากระจก",
          "desc": "ฝึกซ้อม 'รอยยิ้มที่มั่นใจ' ของคุณหน้ากระจกเป็นเวลา 1 นาที",
        },
        {
          "id": "S9",
          "title": "มองแวบเดียว",
          "desc":
              "มองหน้าใครบางคนเป็นเวลา 2 วินาที จากนั้นส่งยิ้มแล้วหันมองไปทางอื่น",
        },
        {
          "id": "S10",
          "title": "คำชมเงียบๆ",
          "desc": "เขียนคอมเมนต์ดีๆ บนโพสต์โซเชียลมีเดียของใครบางคน",
        },
        {
          "id": "S11",
          "title": "แชร์พื้นที่ร่วมกัน",
          "desc":
              "นั่งข้างๆ ใครบางคนในพื้นที่สาธารณะโดยไม่หันมองไปทางอื่นในทันที",
        },
        {
          "id": "S12",
          "title": "มารยาทพื้นฐาน",
          "desc":
              "กล่าวคำว่า 'ขอโทษครับ/ค่ะ' อย่างสุภาพเมื่อต้องเดินเบียดสวนกับใครในทางเดิน",
        },
        {
          "id": "S13",
          "title": "การทักทายที่อบอุ่น",
          "desc":
              "กล่าวคำว่า 'สวัสดีครับ/ค่ะ' กับพนักงานขับรถส่งของหรือบุรุษไปรษณีย์",
        },
        {
          "id": "S14",
          "title": "โบกมือเล็กๆ",
          "desc":
              "โบกมือให้เด็กน้อยหรือสัตว์เลี้ยง (โดยได้รับอนุญาตจากเจ้าของก่อน)",
        },
        {
          "id": "S15",
          "title": "ยิ้มอย่างอ่อนโยน",
          "desc": "ส่งยิ้มอย่างอ่อนโยนให้ผู้คนสามคนที่ไม่ซ้ำกันในวันนี้",
        },
        {
          "id": "S16",
          "title": "ความท้าทายในการสบตา",
          "desc":
              "สบตากับพนักงานเก็บเงินจนกว่าพวกเขาจะเป็นฝ่ายหันมองไปทางอื่นก่อน",
        },
        {
          "id": "S17",
          "title": "ลมหายใจที่สงบ",
          "desc":
              "สูดลมหายใจเข้าลึกๆ 3 ครั้งก่อนที่จะก้าวเข้าสู่พื้นที่สังคมในวันนี้",
        },
        {
          "id": "S18",
          "title": "การอยู่กับปัจจุบัน",
          "desc":
              "ยืนอยู่ในพื้นที่ที่คนพลุกพล่านเป็นเวลา 5 นาทีโดยไม่หยิบโทรศัพท์ขึ้นมาดู",
        },
        {
          "id": "S19",
          "title": "พยักหน้าตามปกติ",
          "desc": "พยักหน้าให้คนแปลกหน้าที่บังเอิญสบตากับคุณ",
        },
        {
          "id": "S20",
          "title": "น้ำเสียงที่นุ่มนวล",
          "desc":
              "กล่าวคำว่า 'ขอให้เป็นวันที่ดีนะครับ/ค่ะ' ให้ใครบางคนในตอนที่คุณกำลังเดินออกจากร้านค้า",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "คำชื่นชม",
          "desc":
              "กล่าวคำชื่นชมอย่างจริงใจให้เพื่อนร่วมงานหรือเพื่อนร่วมชั้นเรียน",
        },
        {
          "id": "SP2",
          "title": "การถามไถ่",
          "desc": "ถามเวลาหรือถามทางจากคนแปลกหน้า",
        },
        {
          "id": "SP3",
          "title": "คุยสัพเพเหระ",
          "desc":
              "ถามใครบางคนว่า 'วันนี้เป็นอย่างไรบ้างครับ/ค่ะ?' แล้วตั้งใจฟังคำตอบ",
        },
        {
          "id": "SP4",
          "title": "การขอความช่วยเหลือ",
          "desc": "เอ่ยปากขอให้พนักงานในร้านช่วยคุณค้นหาสินค้าชิ้นเฉพาะเจาะจง",
        },
        {
          "id": "SP5",
          "title": "ขั้นตอนการสั่ง",
          "desc":
              "สั่งเครื่องดื่มหรืออาหาร แล้วถามพนักงานว่าวันนี้พวกเขาสบายดีไหม",
        },
        {
          "id": "SP6",
          "title": "ทำความรู้จัก",
          "desc": "แนะนำตัวเองให้คนใหม่ๆ ในสภาพแวดล้อมรอบตัวคุณรู้จัก",
        },
        {
          "id": "SP7",
          "title": "คุยเรื่องดินฟ้าอากาศ",
          "desc": "ชวนใครบางคนคุยเรื่องสภาพอากาศในระหว่างที่กำลังยืนรอคิว",
        },
        {
          "id": "SP8",
          "title": "คำถามง่ายๆ",
          "desc":
              "ถามเพื่อนร่วมงานว่า 'สุดสัปดาห์ที่ผ่านมาไปทำอะไรมาบ้างครับ/ค่ะ?'",
        },
        {
          "id": "SP9",
          "title": "เสนอตัวช่วยเหลือ",
          "desc":
              "หากเห็นใครกำลังลำบาก ให้เข้าไปถามว่า 'มีอะไรให้ฉันช่วยไหมครับ/ค่ะ?'",
        },
        {
          "id": "SP10",
          "title": "ขอความเห็น",
          "desc":
              "หยิบสิ่งของชิ้นเล็กๆ ขึ้นมาแล้วถามเพื่อนว่า 'นายคิดอย่างไรกับสิ่งนี้บ้าง?'",
        },
        {
          "id": "SP11",
          "title": "ยืนยันความถูกต้อง",
          "desc":
              "ตรวจสอบความถูกต้องของข้อมูลกับคนแปลกหน้า (เช่น 'ขอโทษครับ คิวนี้ต่อแถวเรื่องนี้ใช่ไหมครับ?')",
        },
        {
          "id": "SP12",
          "title": "ประเมินสภาพแวดล้อม",
          "desc":
              "พูดเปรยๆ เกี่ยวกับสภาพแวดล้อมรอบตัว (เช่น 'วันนี้ที่นี่คนเยอะอัดแน่นจริงๆ เลยนะ')",
        },
        {
          "id": "SP13",
          "title": "ไหว้วานสิ่งเล็กน้อย",
          "desc":
              "ขอให้ใครบางคนบนโต๊ะอาหารช่วยหยิบส่งสิ่งของ (เช่น กระดาษทิชชู่) มาให้คุณ",
        },
        {
          "id": "SP14",
          "title": "คำชมที่อบอุ่น",
          "desc":
              "บอกพนักงานเสิร์ฟก่อนออกจากร้านว่าอาหารมื้อนี้อร่อยและยอดเยี่ยมมาก",
        },
        {
          "id": "SP15",
          "title": "ทักทายตามปกติ",
          "desc":
              "ส่งข้อความคำว่า 'เป็นอย่างไรบ้าง สบายดีไหม?' ไปหาคนที่คุณไม่ได้คุยด้วยมานานเป็นเดือน",
        },
        {
          "id": "SP16",
          "title": "คำถามเปิดกว้าง",
          "desc":
              "ถามใครบางคนว่า 'ในเมืองนี้ สถานที่ท่องเที่ยวที่คุณโปรดปรานที่สุดคือที่ไหนเหรอครับ/ค่ะ?'",
        },
        {
          "id": "SP17",
          "title": "ความเสี่ยงขั้นต่ำ",
          "desc":
              "ถามคนแปลกหน้าว่าพวกเขาพอจะรู้ไหมว่าห้องน้ำที่ใกล้ที่สุดอยู่ตรงไหน",
        },
        {
          "id": "SP18",
          "title": "ชมสิ่งของ",
          "desc":
              "บอกใครบางคนว่าคุณชอบรองเท้า/กระเป๋า/เครื่องประดับของพวกเขาจัง",
        },
        {
          "id": "SP19",
          "title": "การหยุดรออย่างสุภาพ",
          "desc":
              "รอให้คู่สนทนาพูดจบประโยคของเขาอย่างสมบูรณ์ก่อนที่คุณจะเริ่มพูดตอบกลับไป",
        },
        {
          "id": "SP20",
          "title": "การโบกมือบอกลา",
          "desc":
              "โบกมือพร้อมกล่าวคำว่า 'บ๊ายบาย' ให้กับคนที่คุณเพิ่งเสร็จสิ้นการปฏิสัมพันธ์สั้นๆ ด้วย",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "ผู้เสาะหาความคิดเห็น",
          "desc":
              "ถามความคิดเห็นของใครบางคนเกี่ยวกับหนังสือ ภาพยนตร์ หรือบทเพลง",
        },
        {
          "id": "L2",
          "title": "ถามเจาะลึก",
          "desc":
              "ถามคำถามต่อเนื่องเพิ่มเติมหลังจากที่คู่สนทนาเพิ่งเล่าเรื่องราวเกี่ยวกับตัวเองให้คุณฟัง",
        },
        {
          "id": "L3",
          "title": "ขอคำแนะนำ",
          "desc":
              "ถามคนแปลกหน้าเพื่อขอคำแนะนำเกี่ยวกับร้านอาหารอร่อยๆ แถวนี้ที่ควรไปลอง",
        },
        {
          "id": "L4",
          "title": "มองหาจุดร่วม",
          "desc":
              "ค้นหาความสนใจหรืออดิเรกที่ตรงกันระหว่างคุณกับคู่สนทนา แล้วพูดคุยต่อเป็นเวลา 2 นาที",
        },
        {
          "id": "L5",
          "title": "ยื่นมือเข้าช่วย",
          "desc":
              "เสนอตัวเข้าช่วยเหลืองานเล็กๆ น้อยๆ ของใครบางคน (เช่น ช่วยถือถุงสัมภาระ)",
        },
        {
          "id": "L6",
          "title": "จับจุดคุยรอบตัว",
          "desc":
              "เริ่มต้นบทสนทนาโดยหยิบยกเหตุการณ์ที่กำลังเกิดขึ้นรอบตัวคุณทั้งสองคนมาเป็นหัวข้อ",
        },
        {
          "id": "L7",
          "title": "ถามถึงเบื้องหลัง",
          "desc":
              "ถามใครบางคนว่า 'คุณเริ่มเข้าทำงานในสายอาชีพนี้ได้อย่างไรเหรอครับ/ค่ะ?'",
        },
        {
          "id": "L8",
          "title": "ผู้ฟังที่ดี",
          "desc":
              "ตั้งใจฟังสิ่งที่คนอื่นพูดเป็นเวลา 3 นาทีโดยไม่พูดแทรก จากนั้นค่อยสรุปสั้นๆ ในสิ่งที่คุณเข้าใจ",
        },
        {
          "id": "L9",
          "title": "แบ่งปันเสียงหัวเราะ",
          "desc":
              "เล่าเรื่องราวสั้นๆ ที่ตลกขบขันหรือมุกตลกให้กลุ่มคนกลุ่มเล็กๆ ฟัง",
        },
        {
          "id": "L10",
          "title": "ความอยากรู้",
          "desc":
              "ถามใครบางคนว่าพวกเขามาจากจังหวัดไหน และพวกเขารู้สึกชอบอะไรในบ้านเกิดของตนเอง",
        },
        {
          "id": "L11",
          "title": "ความสนใจที่แท้จริง",
          "desc":
              "ถามเพื่อนร่วมงานเกี่ยวกับงานอดิเรกหรือกิจกรรมที่พวกเขาชอบทำนอกเหนือจากเวลางาน",
        },
        {
          "id": "L12",
          "title": "คำแนะนำที่หวังดี",
          "desc":
              "แชร์เคล็ดลับหรือทริกที่เป็นประโยชน์เกี่ยวกับสิ่งที่คุณเชี่ยวชาญให้ผู้อื่นฟัง",
        },
        {
          "id": "L13",
          "title": "การสนับสนุนในกลุ่ม",
          "desc":
              "แสดงความเห็นด้วยกับจุดยืนของใครบางคนในการสนทนากลุ่มเล็กๆ โดยการพยักหน้า",
        },
        {
          "id": "L14",
          "title": "คำชวนสบายๆ",
          "desc":
              "ถามใครบางคนว่า 'สะดวกไปทานมื้อเที่ยงด้วยกันกับพวกเราไหมครับ/ค่ะ?'",
        },
        {
          "id": "L15",
          "title": "สะท้อนความจริงใจ",
          "desc":
              "บอกใครบางคนว่า 'ฉันรู้สึกซาบซึ้งใจจริงๆ ตอนที่คุณช่วยเรื่อง X' พร้อมอธิบายเหตุผล",
        },
        {
          "id": "L16",
          "title": "ไขข้อข้องใจ",
          "desc":
              "ถามใครบางคนว่า 'ฉันสงสัยมาตลอดเลยครับว่า จริงๆ แล้วระบบ X มันทำงานอย่างไรเหรอครับ?'",
        },
        {
          "id": "L17",
          "title": "ผู้เปิดประเด็นกลุ่ม",
          "desc":
              "โยนคำถามในกลุ่มสนทนาที่จำเป็นต้องใช้คนสัก 2-3 คนในการร่วมกันช่วยตอบ",
        },
        {
          "id": "L18",
          "title": "ชมข้อดีในตัว",
          "desc":
              "ชมจุดเด่นด้านนิสัยใจคอของคู่สนทนา (เช่น 'คุณเป็นคนรับฟังคนอื่นได้เก่งมากๆ เลยนะ')",
        },
        {
          "id": "L19",
          "title": "แชร์ประสบการณ์ตรง",
          "desc":
              "พูดประโยคว่า 'ฉันเองก็นึกออกเลย เพราะเคยผ่านสถานการณ์แบบนั้นมาเหมือนกัน' ในระหว่างคุย",
        },
        {
          "id": "L20",
          "title": "ยอมรับความเงียบ",
          "desc":
              "ปล่อยให้ความเงียบเกิดขึ้นในบทสนทนาสักครู่ โดยไม่ต้องรีบร้อนหาคำพูดมาเติมเต็มมัน",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "จุดเริ่มที่กล้าหาญ",
          "desc":
              "ชวนคนที่คุณยังไม่ค่อยสนิทคุ้นเคยชินคุยเพื่อสร้างความสัมพันธ์",
        },
        {
          "id": "ST2",
          "title": "การเปิดเผยอย่างจริงใจ",
          "desc":
              "แชร์เรื่องราวส่วนตัวสั้นๆ หรือแสดงความคิดเห็นของตนเองในท่ามกลางกลุ่มคน",
        },
        {
          "id": "ST3",
          "title": "การถกเถียง",
          "desc":
              "แสดงความเห็นต่างกับผู้อื่นอย่างสุภาพเรียบร้อยพร้อมอธิบายเหตุผลของคุณ",
        },
        {
          "id": "ST4",
          "title": "เข้าร่วมกลุ่มสนทนา",
          "desc":
              "เดินเข้าไปร่วมวงสนทนาที่กำลังดำเนินอยู่ และเสริมประโยคที่มีประโยชน์เข้าไป 1 ประโยค",
        },
        {
          "id": "ST5",
          "title": "ผู้นำประเด็น",
          "desc": "เป็นคนเริ่มต้นเปิดประเด็นคุยเรื่องหัวข้อใหม่ๆ ในกลุ่มสังคม",
        },
        {
          "id": "ST6",
          "title": "คำถามในที่สาธารณะ",
          "desc": "ยกมือขึ้นถามคำถามในการประชุมใหญ่ระดับสาธารณะหรือในห้องเรียน",
        },
        {
          "id": "ST7",
          "title": "คำขอที่ใจกล้า",
          "desc":
              "ถามคนแปลกหน้าในร้านกาแฟหรือสวนสาธารณะว่า 'ขอประทานโทษครับ ฉันขอนั่งตรงนี้ได้ไหมครับ?'",
        },
        {
          "id": "ST8",
          "title": "สะพานเชื่อมสัมพันธ์",
          "desc":
              "แนะนำเพื่อนสองคนที่ไม่รู้จักกันให้ได้รู้จัดกัน พร้อมช่วยหาหัวข้อที่พวกเขาน่าจะสนใจตรงกัน",
        },
        {
          "id": "ST9",
          "title": "ปกป้องสิทธิ์ตัวเอง",
          "desc":
              "บอกคนอื่นอย่างสุภาพให้หยุดพฤติกรรมบางอย่างที่กำลังรบกวนหรือทำให้คุณอึดอัดใจ",
        },
        {
          "id": "ST10",
          "title": "นักเล่าเรื่อง",
          "desc":
              "รับบทบาทเป็นผู้นำในการเล่าเรื่องราวเหตุการณ์หนึ่งให้กลุ่มคนตั้งแต่ 3 คนขึ้นไปฟังจนจบ",
        },
        {
          "id": "ST11",
          "title": "ตั้งข้อสังเกตอย่างมิตร",
          "desc":
              "ตั้งคำถามท้าทายความคิดความเชื่อทั่วไปในกลุ่มด้วยท่าทีที่เป็นมิตรและเคารพผู้อื่น",
        },
        {
          "id": "ST12",
          "title": "ทักทายก่อนเสมอ",
          "desc":
              "เป็นคนแรกที่ส่งเสียงกล่าวคำว่า 'สวัสดีครับ/ค่ะทุกคน!' ทันทีที่ก้าวเท้าเข้าไปในห้อง",
        },
        {
          "id": "ST13",
          "title": "การรับฟังด้วยใจ",
          "desc":
              "ตั้งใจฟังใครบางคนระบายความอัดอั้นหรือความทุกข์ใจ แล้วกล่าวตอบกลับอย่างเห็นอกเห็นใจ",
        },
        {
          "id": "ST14",
          "title": "พูดต่อหน้าชุมชน",
          "desc":
              "พูดนำเสนอความยาว 1-2 นาทีเกี่ยวกับหัวข้อที่คุณหลงใหลในงานพบปะสังสรรค์",
        },
        {
          "id": "ST15",
          "title": "ยอมรับความประหม่า",
          "desc":
              "ยอมรับกับคนในกลุ่มตรงๆ ว่า 'จริงๆ ก่อนหน้านี้ฉันตื่นเต้นมากเลยล่ะ' แล้วหัวเราะไปด้วยกัน",
        },
        {
          "id": "ST16",
          "title": "กำหนดขอบเขต",
          "desc":
              "ปฏิเสธคำเชิญไปงานที่คุณไม่อยากไปอย่างสุภาพ โดยไม่ต้องพยายามหาข้ออ้างมายืดยาว",
        },
        {
          "id": "ST17",
          "title": "ผู้ช่วยไกล่เกลี่ย",
          "desc":
              "ช่วยพูดคุยให้คนสองคนที่กำลังเห็นต่างขัดแย้งกันสามารถหาจุดตรงกลางร่วมกันได้",
        },
        {
          "id": "ST18",
          "title": "ชมเชยต่อหน้าผู้คน",
          "desc":
              "กล่าวชื่นชมความพยายามหรือความสำเร็จของใครบางคนอย่างเปิดเผยต่อหน้ากลุ่มคน",
        },
        {
          "id": "ST19",
          "title": "เข้าหาโดยตรง",
          "desc":
              "เดินเข้าไปเอ่ยปากขอความช่วยเหลือหรือขอคำแนะนำในสิ่งที่คุณจำเป็นต้องใช้จากผู้อื่นตรงๆ",
        },
        {
          "id": "ST20",
          "title": "คุมทิศทางบทสนทนา",
          "desc":
              "ปรับเปลี่ยนทิศทางของเรื่องที่กำลังคุย จากเรื่องที่น่าเบื่อให้กลายมาเป็นเรื่องที่น่าสนใจอย่างลื่นไหล",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "ของขวัญติดมือ",
          "desc":
              "ยื่นขนมหรือของฝากชิ้นเล็กๆ ให้ใครบางคนพร้อมพูดว่า 'เห็นแล้วนึกถึงเลยเอามาฝากครับ/ค่ะ'",
        },
        {
          "id": "B2",
          "title": "ผู้นำแผนงาน",
          "desc":
              "เสนอไอเดียแผนการท่องเที่ยวหรือสถานที่น่าไปพักผ่อนให้กลุ่มเพื่อนกลุ่มเล็กๆ พิจารณา",
        },
        {
          "id": "B3",
          "title": "บอกความสำคัญ",
          "desc":
              "บอกใครบางคนอย่างจริงใจและชัดเจนว่าทำไมคุณถึงรู้สึกโชคดีที่มีพวกเขาเข้ามาอยู่ในชีวิต",
        },
        {
          "id": "B4",
          "title": "เจ้าภาพงานสังคม",
          "desc":
              "เป็นคนนัดแนะชวนเพื่อนๆ สองสามคนออกมาร่วมพบปะหรือจัดเดตนั่งทานกาแฟด้วยกัน",
        },
        {
          "id": "B5",
          "title": "คุยเรื่องลึกซึ้ง",
          "desc":
              "เปิดบทสนทนาพุดคุยแบบเจาะลึกในประเด็นความหมายชีวิตกับคู่สนทนาเป็นเวลานานกว่า 15 นาที",
        },
        {
          "id": "B6",
          "title": "ทลายกำแพงในใจ",
          "desc":
              "เริ่มต้นชวนคุยกับคนที่คุณรู้สึกว่าเขามีบุคลิกน่าเกรงขามหรือคนที่ทำให้คุณรู้สึกประหม่า",
        },
        {
          "id": "B7",
          "title": "กล่าวอวยพรในงาน",
          "desc":
              "อาสายืนขึ้นกล่าวคำชมเชยสั้นๆ หรือกล่าวคำอวยพรร่วมยินดีให้ใครบางคนในงานเลี้ยง",
        },
        {
          "id": "B8",
          "title": "กล้าปฏิเสธ",
          "desc":
              "กล่าวปฏิเสธคำขอร้องที่ไม่สมเหตุสมผลอย่างหนักแน่นแต่สุภาพ โดยไม่พยายามแต่งเรื่องอธิบาย",
        },
        {
          "id": "B9",
          "title": "เทียบเชิญผู้ใหญ่",
          "desc":
              "ติดต่อหาบุคคลรุ่นพี่ที่คุณชื่นชมเพื่อขอเวลาพูดคุยสั้นๆ 10 นาทีเพื่อขอคำชี้แนะเชิงแนวคิด",
        },
        {
          "id": "B10",
          "title": "เปิดใจคุยเรื่องอารมณ์",
          "desc":
              "เป็นฝ่ายเริ่มต้นชวนเพื่อนสนิทคุยเรื่องความรู้สึกข้างในใจหรือปัญหาสุขภาพจิตอย่างจริงจัง",
        },
        {
          "id": "B11",
          "title": "ผู้ประสานรอยร้าว",
          "desc":
              "ช่วยพูดคุยเชื่อมประสานให้คนสองคนที่กำลังตึงเครียดหรือผิดใจกันหันหน้ากลับมาคุยกันดีๆ",
        },
        {
          "id": "B12",
          "title": "เอ่ยชมอย่างกล้าหาญ",
          "desc":
              "เดินเข้าไปหาคนแปลกหน้าที่เดินสวนกันแล้วเอ่ยปากชมสิ่งที่คุณรู้สึกประทับใจในตัวเขาตรงๆ",
        },
        {
          "id": "B13",
          "title": "ขยายคอนเนกชัน",
          "desc":
              "แนะนำตัวเองกับบุคคลระดับมืออาชีพในสายงานที่คุณตั้งเป้าหมายไว้เพื่อขอคำแนะนำด้านอาชีพ",
        },
        {
          "id": "B14",
          "title": "ความจริงที่ต้องกล้า",
          "desc":
              "พูดความจริงในเรื่องที่พูดยากแต่ส่งผลดีต่อระยะยาวในความสัมพันธ์ให้คู่กรณีฟังอย่างจริงใจ",
        },
        {
          "id": "B15",
          "title": "เจ้าบ้านที่สมบูรณ์",
          "desc":
              "จัดงานปาร์ตี้เล็กๆ ของตัวเองและดูแลใส่ใจให้แขกทุกคนที่มางานรู้สึกอบอุ่นได้รับการต้อนรับ",
        },
        {
          "id": "B16",
          "title": "ผู้นำแถลง",
          "desc":
              "เสนอตัวเป็นผู้ดำเนินรายการหรือรับหน้าที่นำจัดกิจกรรมในช่วงใดช่วงหนึ่งของการประชุม",
        },
        {
          "id": "B17",
          "title": "เล่าปมเพื่อหนุนใจ",
          "desc":
              "บอกเล่าเรื่องราวความล้มเหลวหรือบาดแผลในอดีตที่คุณก้าวข้ามมาได้เพื่อสร้างแรงบันดาลใจให้ผู้อื่น",
        },
        {
          "id": "B18",
          "title": "ขอโทษอย่างจริงใจ",
          "desc":
              "ติดต่อไปหาใครบางคนเพื่อกล่าวคำขอโทษในความผิดพลาดที่คุณเคยทำไว้ในอดีต แม้เรื่องจะผ่านมานานแล้ว",
        },
        {
          "id": "B19",
          "title": "บทบาทที่ปรึกษา",
          "desc":
              "เสนอตัวเข้าช่วยแนะนำแชร์ความรู้เชิงทักษะให้กับรุ่นน้องที่มีประสบการณ์น้อยกว่าคุณ",
        },
        {
          "id": "B20",
          "title": "นักสร้างสรรค์สังคม",
          "desc":
              "ริเริ่มสร้างธรรมเนียมปฏิบัติกิจกรรมใหม่ๆ ร่วมกันในกลุ่มเพื่อน (เช่น วันนัดกินข้าวประจำสัปดาห์)",
        },
      ],
    },
    'vi': {
      "Seedling": [
        {
          "id": "S1",
          "title": "Bước đi đầu tiên",
          "desc":
              "Hãy giao tiếp bằng mắt và mỉm cười với một người bất kỳ trong ngày hôm nay.",
        },
        {
          "id": "S2",
          "title": "Một lời chào đơn giản",
          "desc":
              "Hãy nói 'Chào buổi sáng' hoặc 'Xin chào' với một người hàng xóm.",
        },
        {
          "id": "S3",
          "title": "Lời cảm ơn",
          "desc": "Hãy nói 'Cảm ơn' thật rõ ràng với một người chủ cửa hàng.",
        },
        {
          "id": "S4",
          "title": "Sự quan sát",
          "desc": "Hãy nhận ra một điều tích cực ở một người lạ và mỉm cười.",
        },
        {
          "id": "S5",
          "title": "Cú vẫy tay lặng lẽ",
          "desc": "Hãy vẫy tay chào một người quen mà bạn nhìn thấy từ xa.",
        },
        {
          "id": "S6",
          "title": "Giữ cửa",
          "desc": "Hãy giữ cửa mở cho một người đang đi ngay phía sau bạn.",
        },
        {
          "id": "S7",
          "title": "Cái gật đầu",
          "desc":
              "Hãy gật đầu thân thiện với một người đồng nghiệp khi bạn đi lướt qua họ.",
        },
        {
          "id": "S8",
          "title": "Tập trước gương",
          "desc":
              "Hãy luyện tập 'nụ cười tự tin' của bạn trước gương trong vòng 1 phút.",
        },
        {
          "id": "S9",
          "title": "Ánh nhìn thoáng qua",
          "desc":
              "Hãy nhìn ai đó trong 2 giây, sau đó mỉm cười và quay mặt đi hướng khác.",
        },
        {
          "id": "S10",
          "title": "Lời khen thầm lặng",
          "desc":
              "Hãy viết một bình luận tử tế trên một bài đăng mạng xã hội của ai đó.",
        },
        {
          "id": "S11",
          "title": "Chia sẻ không gian",
          "desc":
              "Hãy ngồi cạnh ai đó ở khu vực công cộng mà không lập tức quay mặt đi nơi khác.",
        },
        {
          "id": "S12",
          "title": "Sự lịch thiệp đơn giản",
          "desc":
              "Hãy nói 'Xin lỗi' một cách lịch sự khi đi lướt qua ai đó ở hành lang.",
        },
        {
          "id": "S13",
          "title": "Lời chào ấm áp",
          "desc": "Hãy nói 'Xin chào' với một nhân viên giao hàng hoặc bưu tá.",
        },
        {
          "id": "S14",
          "title": "Cú vẫy tay nhỏ",
          "desc":
              "Hãy vẫy tay chào một em bé hoặc một thú cưng (sau khi được chủ nhân cho phép).",
        },
        {
          "id": "S15",
          "title": "Nụ cười nhẹ nhàng",
          "desc": "Hãy mỉm cười với ba người khác nhau trong ngày hôm nay.",
        },
        {
          "id": "S16",
          "title": "Thử thách ánh nhìn",
          "desc":
              "Hãy giữ giao tiếp bằng mắt với nhân viên thu ngân cho đến khi họ là người quay đi trước.",
        },
        {
          "id": "S17",
          "title": "Hơi thở bình tâm",
          "desc":
              "Hãy hít thở thật sâu 3 lần trước khi bước vào một không gian xã hội ngày hôm nay.",
        },
        {
          "id": "S18",
          "title": "Sự hiện diện",
          "desc":
              "Hãy đứng ở một khu vực đông người trong 5 phút mà không nhìn vào điện thoại của bạn.",
        },
        {
          "id": "S19",
          "title": "Cái gật đầu ngẫu nhiên",
          "desc":
              "Hãy gật đầu chào một người lạ có giao tiếp bằng mắt với bạn.",
        },
        {
          "id": "S20",
          "title": "Giọng nói nhẹ nhàng",
          "desc":
              "Hãy nói 'Chúc một ngày tốt lành' với ai đó khi bạn rời khỏi cửa hàng.",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "Lời khen ngợi",
          "desc":
              "Hãy dành một lời khen chân thành cho một đồng nghiệp hoặc bạn cùng lớp.",
        },
        {
          "id": "SP2",
          "title": "Hỏi thăm",
          "desc": "Hãy hỏi một người lạ về thời gian hoặc chỉ đường.",
        },
        {
          "id": "SP3",
          "title": "Trò chuyện xã giao",
          "desc":
              "Hãy hỏi ai đó 'Ngày hôm nay của bạn thế nào?' và lắng nghe câu trả lời.",
        },
        {
          "id": "SP4",
          "title": "Lời yêu cầu",
          "desc":
              "Hãy nhờ một nhân viên cửa hàng giúp đỡ để tìm một món đồ cụ thể.",
        },
        {
          "id": "SP5",
          "title": "Gọi món",
          "desc":
              "Hãy gọi một món đồ uống hoặc thức ăn và hỏi thăm nhân viên xem họ thế nào.",
        },
        {
          "id": "SP6",
          "title": "Làm quen",
          "desc":
              "Hãy tự giới thiệu bản thân với một người mới trong khu vực của bạn.",
        },
        {
          "id": "SP7",
          "title": "Chuyện thời tiết",
          "desc":
              "Hãy nhắc đến chuyện thời tiết với ai đó trong khi đang đứng xếp hàng chờ.",
        },
        {
          "id": "SP8",
          "title": "Lời hỏi thăm đơn giản",
          "desc": "Hãy hỏi một đồng nghiệp 'Bạn đã làm gì vào cuối tuần qua?'",
        },
        {
          "id": "SP9",
          "title": "Lời đề nghị giúp đỡ",
          "desc":
              "Hãy hỏi ai đó 'Bạn có cần giúp một tay không?' nếu trông họ có vẻ đang gặp khó khăn.",
        },
        {
          "id": "SP10",
          "title": "Hỏi ý kiến",
          "desc":
              "Hãy hỏi một người bạn 'Bạn nghĩ sao về cái này?' đối với một đồ vật nhỏ.",
        },
        {
          "id": "SP11",
          "title": "Xác nhận",
          "desc":
              "Hãy xác nhận một chi tiết với người lạ (ví dụ: 'Đây có phải hàng xếp đúng không ạ?').",
        },
        {
          "id": "SP12",
          "title": "Không gian chung",
          "desc":
              "Hãy đưa ra một nhận xét nhỏ về môi trường xung quanh (ví dụ: 'Ở đây thực sự đông đúc quá').",
        },
        {
          "id": "SP13",
          "title": "Nhờ vả nhỏ",
          "desc":
              "Hãy nhờ ai đó chuyển giúp bạn một món đồ (như khăn ăn) khi ngồi ở bàn ăn.",
        },
        {
          "id": "SP14",
          "title": "Phản hồi ấm áp",
          "desc":
              "Hãy khen ngợi người phục vụ rằng đồ ăn rất tuyệt trước khi rời đi.",
        },
        {
          "id": "SP15",
          "title": "Hỏi thăm thông thường",
          "desc":
              "Hãy gửi một tin nhắn 'Dạo này thế nào?' cho một người bạn đã không nói chuyện cả tháng qua.",
        },
        {
          "id": "SP16",
          "title": "Câu hỏi mở",
          "desc":
              "Hãy hỏi ai đó 'Địa điểm yêu thích nhất của bạn để ghé thăm ở thành phố này là gì?'",
        },
        {
          "id": "SP17",
          "title": "Rủi ro tối thiểu",
          "desc":
              "Hãy hỏi một người lạ xem họ có biết nhà vệ sinh gần nhất ở đâu không.",
        },
        {
          "id": "SP18",
          "title": "Khen ngợi đồ vật",
          "desc":
              "Nhận xét với ai đó rằng bạn thích đôi giày/túi xách/phụ kiện của họ.",
        },
        {
          "id": "SP19",
          "title": "Sự tạm dừng lịch sự",
          "desc":
              "Hãy chờ cho đối phương nói xong hoàn toàn câu chuyện trước khi bạn phản hồi lại họ.",
        },
        {
          "id": "SP20",
          "title": "Cú vẫy tay thân thiện",
          "desc":
              "Hãy vẫy tay và nói 'Tạm biệt' với một người mà bạn vừa mới có một tương tác ngắn.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "Người tìm kiếm ý kiến",
          "desc":
              "Hãy hỏi ý kiến của một ai đó về một cuốn sách, bộ phim hoặc một bài hát.",
        },
        {
          "id": "L2",
          "title": "Chi tiết nhỏ",
          "desc":
              "Hãy đặt một câu hỏi đào sâu thêm sau khi ai đó kể cho bạn nghe điều gì đó về bản thân họ.",
        },
        {
          "id": "L3",
          "title": "Lời gợi ý",
          "desc":
              "Hãy nhờ một người lạ gợi ý về một địa điểm ăn uống ngon ở khu vực gần đây.",
        },
        {
          "id": "L4",
          "title": "Sự kết nối",
          "desc":
              "Hãy tìm ra một sở thích chung với ai đó và cùng trò chuyện về nó trong vòng 2 phút.",
        },
        {
          "id": "L5",
          "title": "Bàn tay trợ giúp",
          "desc":
              "Hãy đề nghị giúp đỡ ai đó một việc nhỏ (chẳng hạn như xách hộ một chiếc túi).",
        },
        {
          "id": "L6",
          "title": "Quan sát xã hội",
          "desc":
              "Hãy bắt đầu một cuộc trò chuyện dựa trên một điều gì đó đang xảy ra xung quanh cả hai người.",
        },
        {
          "id": "L7",
          "title": "Câu hỏi mở sâu sắc",
          "desc":
              "Hãy hỏi ai đó 'Làm thế nào mà bạn lại bước chân vào ngành nghề công việc này?'",
        },
        {
          "id": "L8",
          "title": "Người lắng nghe tích cực",
          "desc":
              "Hãy lắng nghe ai đó nói trong vòng 3 phút mà không ngắt lời, sau đó tóm tắt lại những gì họ vừa chia sẻ.",
        },
        {
          "id": "L9",
          "title": "Tiếng cười chung",
          "desc":
              "Hãy kể một câu chuyện ngắn, hài hước hoặc một câu chuyện cười cho một nhóm nhỏ.",
        },
        {
          "id": "L10",
          "title": "Sự hiếu kỳ",
          "desc":
              "Hãy hỏi ai đó xem họ đến từ đâu và họ thích điều gì nhất ở quê hương của mình.",
        },
        {
          "id": "L11",
          "title": "Mối quan tâm chân thành",
          "desc":
              "Hãy hỏi một người đồng nghiệp về những sở thích của họ ngoài giờ làm việc.",
        },
        {
          "id": "L12",
          "title": "Lời khuyên khéo léo",
          "desc":
              "Hãy đưa ra một mẹo hữu ích cho ai đó về một lĩnh vực hay kỹ năng mà bạn làm tốt.",
        },
        {
          "id": "L13",
          "title": "Cái gật đầu nhóm",
          "desc":
              "Hãy thể hiện sự đồng tình với quan điểm của ai đó trong một cuộc thảo luận nhóm nhỏ.",
        },
        {
          "id": "L14",
          "title": "Lời mời thân mật",
          "desc":
              "Hãy hỏi ai đó 'Bạn có muốn đi ăn trưa cùng với tụi mình không?'",
        },
        {
          "id": "L15",
          "title": "Sự phản hồi thành thật",
          "desc":
              "Hãy nói với ai đó 'Mình thực sự trân trọng khi bạn đã làm việc X' và giải thích rõ lý do.",
        },
        {
          "id": "L16",
          "title": "Khám phá ẩn số",
          "desc":
              "Hãy hỏi ai đó 'Mình đã luôn thắc mắc, thực ra thì cơ chế X hoạt động như thế nào vậy nhỉ?'",
        },
        {
          "id": "L17",
          "title": "Dẫn dắt nhóm nhỏ",
          "desc":
              "Hãy đặt ra một câu hỏi đòi hỏi khoảng 2 hoặc 3 người trong nhóm cùng thảo luận trả lời.",
        },
        {
          "id": "L18",
          "title": "Lời khen chân thực",
          "desc":
              "Hãy khen ngợi ai đó về một nét tính cách đặc trưng (ví dụ: 'Bạn là một người lắng nghe rất tuyệt vời').",
        },
        {
          "id": "L19",
          "title": "Trải nghiệm chia sẻ",
          "desc":
              "Hãy nói câu 'Mình cũng từng ở trong hoàn cảnh tương tự như vậy rồi' trong một cuộc đối thoại.",
        },
        {
          "id": "L20",
          "title": "Khoảng lặng ý nghĩa",
          "desc":
              "Hãy chấp nhận một khoảng lặng xuất hiện trong cuộc trò chuyện mà không vội vã tìm lời nói lấp đầy nó.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "Khởi đầu dũng cảm",
          "desc":
              "Hãy chủ động bắt chuyện với một người mà bạn chưa quen biết nhiều.",
        },
        {
          "id": "ST2",
          "title": "Chia sẻ chân thật",
          "desc":
              "Hãy chia sẻ một câu chuyện cá nhân ngắn hoặc bày tỏ quan điểm trong môi trường tập thể.",
        },
        {
          "id": "ST3",
          "title": "Cuộc thảo luận",
          "desc":
              "Hãy từ chối hoặc phản biện quan điểm của ai đó một cách lịch sự và giải thích lý do.",
        },
        {
          "id": "ST4",
          "title": "Gia nhập nhóm",
          "desc":
              "Hãy tham gia vào một cuộc trò chuyện nhóm đang diễn ra và đóng góp một câu nói có chiều sâu.",
        },
        {
          "id": "ST5",
          "title": "Mở đầu chủ đề",
          "desc":
              "Hãy đưa ra một chủ đề trò chuyện hoàn toàn mới trong một nhóm xã hội.",
        },
        {
          "id": "ST6",
          "title": "Câu hỏi công khai",
          "desc":
              "Hãy giơ tay đặt câu hỏi trong một cuộc họp công khai hoặc trong không gian lớp học.",
        },
        {
          "id": "ST7",
          "title": "Yêu cầu táo bạo",
          "desc":
              "Hãy hỏi một người lạ ở quán cà phê hoặc công viên xem bạn có thể ngồi cạnh họ được không.",
        },
        {
          "id": "ST8",
          "title": "Cầu nối trò chuyện",
          "desc":
              "Hãy giới thiệu hai người chưa quen biết nhau làm quen, và tìm ra điểm chung để kết nối họ.",
        },
        {
          "id": "ST9",
          "title": "Đòi hỏi quyết đoán",
          "desc":
              "Hãy lịch sự yêu cầu ai đó di chuyển vị trí hoặc dừng một hành động đang làm phiền bạn.",
        },
        {
          "id": "ST10",
          "title": "Người kể chuyện",
          "desc":
              "Hãy đóng vai trò chủ đạo trong việc kể một câu chuyện cho một nhóm từ 3 người trở lên nghe.",
        },
        {
          "id": "ST11",
          "title": "Thử thách cởi mở",
          "desc":
              "Hãy phản biện một ý kiến phổ biến trong nhóm bằng một thái độ thân thiện và tôn trọng.",
        },
        {
          "id": "ST12",
          "title": "Chủ động phá băng",
          "desc":
              "Hãy trở thành người đầu tiên chủ động cất tiếng chào 'Xin chào mọi người!' khi bước vào phòng.",
        },
        {
          "id": "ST13",
          "title": "Lắng nghe thấu cảm",
          "desc":
              "Hãy kiên nhẫn lắng nghe ai đó chút bầu tâm sự hoặc xả cơn giận, rồi đưa ra phản hồi mang tính hỗ trợ.",
        },
        {
          "id": "ST14",
          "title": "Phát biểu trước đám đông",
          "desc":
              "Hãy nói chuyện từ 1-2 phút về một chủ đề bạn cực kỳ đam mê tại một buổi tụ họp xã hội.",
        },
        {
          "id": "ST15",
          "title": "Bộc lộ sự yếu đuối",
          "desc":
              "Hãy thú nhận trước một nhóm rằng thực ra bạn đã rất lo lắng về điều gì đó, rồi cùng cười xòa.",
        },
        {
          "id": "ST16",
          "title": "Thiết lập ranh giới",
          "desc":
              "Hãy lịch sự từ chối một lời mời mà bạn không muốn tham gia mà không cần đưa ra lý do dài dòng.",
        },
        {
          "id": "ST17",
          "title": "Hòa giải tích cực",
          "desc":
              "Hãy giúp hai người đang có bất đồng tìm ra một điểm chung hòa giải.",
        },
        {
          "id": "ST18",
          "title": "Khen ngợi công khai",
          "desc":
              "Hãy công khai tán dương nỗ lực hoặc thành tựu của ai đó trước mặt cả nhóm.",
        },
        {
          "id": "ST19",
          "title": "Tiếp cận trực tiếp",
          "desc":
              "Hãy trực tiếp hỏi xin một sự giúp đỡ hoặc một lời khuyên thiết thực mà bạn cần từ ai đó.",
        },
        {
          "id": "ST20",
          "title": "Chuyển hướng cuộc trò chuyện",
          "desc":
              "Hãy bẻ lái hướng trò chuyện một cách mượt mà từ một chủ đề tẻ nhạt sang một lĩnh vực thú vị hơn.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "Món quà nhỏ",
          "desc":
              "Hãy đưa một chút đồ ăn nhẹ hoặc món quà nhỏ cho ai đó và nói 'Mình nghĩ bạn sẽ thích cái này'.",
        },
        {
          "id": "B2",
          "title": "Sự dẫn dắt táo bạo",
          "desc":
              "Hãy chủ động đề xuất một kế hoạch đi chơi hoặc một địa điểm tham quan cho một nhóm bạn nhỏ.",
        },
        {
          "id": "B3",
          "title": "Bày tỏ lòng tri ân",
          "desc":
              "Hãy nói với một người một cách cụ thể lý do tại sao bạn cảm thấy may mắn khi có họ trong đời.",
        },
        {
          "id": "B4",
          "title": "Người kết nối",
          "desc":
              "Hãy đứng ra hẹn gặp hoặc tổ chức một buổi hẹn cà phê nhỏ cho một vài người bạn.",
        },
        {
          "id": "B5",
          "title": "Cuộc đối thoại sâu sắc",
          "desc":
              "Hãy có một cuộc trò chuyện sâu sắc, mang tính bản chất với ai đó kéo dài hơn 15 phút.",
        },
        {
          "id": "B6",
          "title": "Đỉnh cao tự tin",
          "desc":
              "Hãy chủ động bắt chuyện với một người mà bạn cảm thấy có phần e dè hoặc ngại tiếp cận.",
        },
        {
          "id": "B7",
          "title": "Lời chúc mừng công khai",
          "desc":
              "Hãy đứng lên phát biểu một lời chúc rượu ngắn, tích cực hoặc lời tán dương cho ai đó trong buổi tiệc.",
        },
        {
          "id": "B8",
          "title": "Người vạch ranh giới",
          "desc":
              "Hãy nói 'Không' trước một lời yêu cầu vô lý một cách dứt khoát nhưng lịch sự, không tìm lý do ngụy biện.",
        },
        {
          "id": "B9",
          "title": "Lời đề nghị trực tiếp",
          "desc":
              "Hãy liên hệ với một tiền bối bạn ngưỡng mộ để xin 10 phút trò chuyện ngắn xin lời khuyên định hướng.",
        },
        {
          "id": "B10",
          "title": "Dẫn dắt cảm xúc",
          "desc":
              "Hãy là người chủ động mở lời trò chuyện với bạn thân về cảm xúc bên trong hoặc sức khỏe tinh thần.",
        },
        {
          "id": "B11",
          "title": "Người hòa giải xã hội",
          "desc":
              "Hãy giúp hai người đang chiến tranh lạnh hoặc xung đột nhỏ ngồi lại nói chuyện bình tĩnh để làm hòa.",
        },
        {
          "id": "B12",
          "title": "Lời khen táo bạo",
          "desc":
              "Hãy đi đến trước mặt một người hoàn toàn xa lạ và nói thẳng ra một điểm bạn thực sự ngưỡng mộ ở họ.",
        },
        {
          "id": "B13",
          "title": "Mở rộng vòng kết nối",
          "desc":
              "Hãy chủ động tự giới thiệu bản thân với một chuyên gia trong ngành nghề mục tiêu của bạn để xin lời khuyên nghề nghiệp.",
        },
        {
          "id": "B14",
          "title": "Sự thật dũng cảm",
          "desc":
              "Hãy nói một sự thật tuy khó nghe nhưng mang lại lợi ích lâu dài cho mối quan hệ với đối phương.",
        },
        {
          "id": "B15",
          "title": "Chủ nhà chu đáo",
          "desc":
              "Hãy tự mình đứng ra tổ chức một sự kiện giao lưu nhỏ và chăm sóc sao cho mọi vị khách đều cảm thấy thoải mái.",
        },
        {
          "id": "B16",
          "title": "Người điều phối",
          "desc":
              "Hãy xung phong làm người dẫn chương trình hoặc chịu trách nhiệm dẫn dắt một phần nhỏ của cuộc họp.",
        },
        {
          "id": "B17",
          "title": "Chia sẻ để truyền cảm hứng",
          "desc":
              "Hãy kể câu chuyện về thất bại hay khó khăn trong quá khứ mà bạn đã vượt qua để tiếp thêm động lực cho người khác.",
        },
        {
          "id": "B18",
          "title": "Lời xin lỗi chân thành",
          "desc":
              "Hãy chủ động liên hệ với ai đó để xin lỗi về một sai lầm bạn từng mắc phải trong quá khứ, dù chuyện đã qua lâu.",
        },
        {
          "id": "B19",
          "title": "Vai trò người hướng dẫn",
          "desc":
              "Hãy chủ động đề nghị hỗ trợ, chia sẻ kinh nghiệm kỹ năng cốt lõi cho một người cấp dưới chưa có nhiều kinh nghiệm.",
        },
        {
          "id": "B20",
          "title": "Kiến trúc sư xã hội",
          "desc":
              "Hãy khởi xướng một truyền thống sinh hoạt cố định mới cho nhóm bạn (ví dụ: ngày hẹn ăn tối cố định mỗi tuần).",
        },
      ],
    },
    'id': {
      "Seedling": [
        {
          "id": "S1",
          "title": "Langkah Pertama",
          "desc":
              "Lakukan kontak mata dan tersenyumlah pada satu orang hari ini.",
        },
        {
          "id": "S2",
          "title": "Sapaan Sederhana",
          "desc": "Ucapkan 'Selamat pagi' atau 'Halo' kepada tetangga.",
        },
        {
          "id": "S3",
          "title": "Ucapan Terima Kasih",
          "desc": "Ucapkan 'Terima kasih' dengan jelas kepada penjaga toko.",
        },
        {
          "id": "S4",
          "title": "Pengamatan",
          "desc":
              "Perhatikan sesuatu yang positif tentang orang asing dan tersenyumlah.",
        },
        {
          "id": "S5",
          "title": "Lambaian Diam-diam",
          "desc": "Melambailah pada seseorang yang kamu kenali dari kejauhan.",
        },
        {
          "id": "S6",
          "title": "Menahan Pintu",
          "desc":
              "Tahan pintu agar tetap terbuka untuk seseorang di belakangmu.",
        },
        {
          "id": "S7",
          "title": "Anggukan",
          "desc":
              "Berikan anggukan ramah kepada rekan kerja saat kamu berpapasan dengan mereka.",
        },
        {
          "id": "S8",
          "title": "Cermin",
          "desc":
              "Latihlah 'senyum percaya diri' kamu di depan cermin selama 1 menit.",
        },
        {
          "id": "S9",
          "title": "Pandangan Sekilas",
          "desc":
              "Pandang seseorang selama 2 detik, lalu tersenyum dan buang muka.",
        },
        {
          "id": "S10",
          "title": "Pujian Diam-diam",
          "desc":
              "Tulis komentar yang baik pada unggahan media sosial seseorang.",
        },
        {
          "id": "S11",
          "title": "Berbagi Ruang",
          "desc":
              "Duduklah di sebelah seseorang di area publik tanpa langsung membuang muka.",
        },
        {
          "id": "S12",
          "title": "Penghargaan Sederhana",
          "desc":
              "Ucapkan 'Permisi' dengan sopan saat berpapasan dengan seseorang di lorong.",
        },
        {
          "id": "S13",
          "title": "Sapaan Hangat",
          "desc": "Ucapkan 'Hai' kepada kurir pengiriman atau petugas pos.",
        },
        {
          "id": "S14",
          "title": "Lambaian Kecil",
          "desc":
              "Melambailah pada anak kecil atau hewan peliharaan (dengan izin pemiliknya).",
        },
        {
          "id": "S15",
          "title": "Senyuman Lembut",
          "desc": "Tersenyumlah pada tiga orang yang berbeda hari ini.",
        },
        {
          "id": "S16",
          "title": "Tantangan Kontak Mata",
          "desc":
              "Pertahankan kontak mata dengan kasir sampai mereka membuang muka terlebih dahulu.",
        },
        {
          "id": "S17",
          "title": "Napas Tenang",
          "desc":
              "Tarik 3 napas dalam-dalam sebelum memasuki ruang sosial hari ini.",
        },
        {
          "id": "S18",
          "title": "Kehadiran",
          "desc":
              "Berdirilah di area yang ramai selama 5 menit tanpa melihat ponselmu.",
        },
        {
          "id": "S19",
          "title": "Anggukan Kasual",
          "desc":
              "Mengangguklah kepada orang asing yang melakukan kontak mata denganmu.",
        },
        {
          "id": "S20",
          "title": "Suara Lembut",
          "desc":
              "Ucapkan 'Semoga harimu menyenangkan' kepada seseorang saat kamu meninggalkan toko.",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "Pujian",
          "desc":
              "Berikan pujian yang tulus kepada rekan kerja atau teman sekelas.",
        },
        {
          "id": "SP2",
          "title": "Pertanyaan",
          "desc": "Tanyakan jam atau arah jalan kepada orang asing.",
        },
        {
          "id": "SP3",
          "title": "Basa-basi",
          "desc":
              "Tanyakan kepada seseorang 'Bagaimana harimu berjalan?' dan dengarkan jawabannya.",
        },
        {
          "id": "SP4",
          "title": "Permintaan",
          "desc":
              "Mintalah bantuan karyawan toko untuk menemukan barang tertentu.",
        },
        {
          "id": "SP5",
          "title": "Pesanan",
          "desc":
              "Pesan minuman atau makanan dan tanyakan kepada staf bagaimana keadaan mereka.",
        },
        {
          "id": "SP6",
          "title": "Perkenalan",
          "desc":
              "Perkenalkan dirimu kepada seseorang yang baru di lingkunganmu.",
        },
        {
          "id": "SP7",
          "title": "Obrolan Cuaca",
          "desc": "Sebutkan tentang cuaca kepada seseorang saat mengantre.",
        },
        {
          "id": "SP8",
          "title": "Pertanyaan Sederhana",
          "desc":
              "Tanyakan kepada rekan kerja 'Apa yang kamu lakukan selama akhir pekan?'",
        },
        {
          "id": "SP9",
          "title": "Tawaran Bantuan",
          "desc":
              "Tanyakan kepada seseorang 'Apakah kamu butuh bantuan dengan itu?' jika mereka terlihat kesulitan.",
        },
        {
          "id": "SP10",
          "title": "Pendapat",
          "desc":
              "Tanyakan kepada seorang teman 'Bagaimana menurutmu tentang ini?' mengenai suatu objek kecil.",
        },
        {
          "id": "SP11",
          "title": "Konfirmasi",
          "desc":
              "Konfirmasikan detail dengan orang asing (misalnya, 'Apakah ini antrean yang benar?').",
        },
        {
          "id": "SP12",
          "title": "Ruang Bersama",
          "desc":
              "Berikan komentar kecil tentang lingkungan sekitar (misalnya, 'Tempat ini sangat ramai').",
        },
        {
          "id": "SP13",
          "title": "Bantuan Kecil",
          "desc":
              "Mintalah seseorang untuk mengambilkan sesuatu (seperti tisu) di meja.",
        },
        {
          "id": "SP14",
          "title": "Umpan Balik Hangat",
          "desc":
              "Beritahu pelayan bahwa makanannya sangat enak sebelum pergi.",
        },
        {
          "id": "SP15",
          "title": "Kabar Kasual",
          "desc":
              "Kirim pesan 'Bagaimana kabarmu?' kepada seseorang yang sudah sebulan tidak kamu ajak bicara.",
        },
        {
          "id": "SP16",
          "title": "Pertanyaan Terbuka",
          "desc":
              "Tanyakan kepada seseorang 'Di mana tempat favoritmu untuk dikunjungi di kota ini?'",
        },
        {
          "id": "SP17",
          "title": "Risiko Terkecil",
          "desc":
              "Tanyakan kepada orang asing apakah mereka tahu di mana toilet terdekat.",
        },
        {
          "id": "SP18",
          "title": "Pujian Barang",
          "desc":
              "Beritahu seseorang bahwa kamu menyukai sepatu/tas/aksesori mereka.",
        },
        {
          "id": "SP19",
          "title": "Jeda Sopan",
          "desc":
              "Tunggulah sampai seseorang selesai berbicara sepenuhnya sebelum menanggapi mereka.",
        },
        {
          "id": "SP20",
          "title": "Lambaian Ramah",
          "desc":
              "Melambailah dan ucapkan 'Dada' kepada seseorang yang baru saja berinteraksi singkat denganmu.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "Pencari Pendapat",
          "desc": "Tanyakan pendapat seseorang tentang buku, film, atau lagu.",
        },
        {
          "id": "L2",
          "title": "Detail",
          "desc":
              "Ajukan pertanyaan lanjutan setelah seseorang menceritakan sesuatu tentang dirinya kepadamu.",
        },
        {
          "id": "L3",
          "title": "Rekomendasi",
          "desc":
              "Mintalah rekomendasi dari orang asing tentang tempat makan yang enak di sekitar sini.",
        },
        {
          "id": "L4",
          "title": "Koneksi",
          "desc":
              "Temukan minat yang sama dengan seseorang dan bicarakan hal itu selama 2 menit.",
        },
        {
          "id": "L5",
          "title": "Uluran Tangan",
          "desc":
              "Tawarkan diri untuk membantu seseorang dengan tugas kecil (seperti membawakan tas).",
        },
        {
          "id": "L6",
          "title": "Pengamatan Sosial",
          "desc":
              "Mulailah percakapan berdasarkan sesuatu yang terjadi di sekitar kalian berdua.",
        },
        {
          "id": "L7",
          "title": "Pertanyaan Terbuka",
          "desc":
              "Tanyakan kepada seseorang 'Bagaimana awal mulanya kamu terjun ke bidang pekerjaan ini?'",
        },
        {
          "id": "L8",
          "title": "Pendengar Aktif",
          "desc":
              "Dengarkan seseorang selama 3 menit tanpa memotong, lalu rangkum apa yang mereka katakan.",
        },
        {
          "id": "L9",
          "title": "Tawa Bersama",
          "desc":
              "Ceritakan cerita pendek yang lucu atau lelucon kepada kelompok kecil.",
        },
        {
          "id": "L10",
          "title": "Rasa Ingin Tahu",
          "desc":
              "Tanyakan kepada seseorang dari mana mereka berasal dan apa yang mereka sukai dari tempat itu.",
        },
        {
          "id": "L11",
          "title": "Minat Tulus",
          "desc":
              "Tanyakan kepada rekan kerja tentang hobi mereka di luar pekerjaan.",
        },
        {
          "id": "L12",
          "title": "Saran Halus",
          "desc":
              "Berikan tips bermanfaat kepada seseorang tentang sesuatu yang kamu kuasai.",
        },
        {
          "id": "L13",
          "title": "Anggukan Kelompok",
          "desc": "Setujui pendapat seseorang dalam diskusi kelompok kecil.",
        },
        {
          "id": "L14",
          "title": "Undangan Kasual",
          "desc":
              "Tanyakan kepada seseorang 'Apakah kamu mau ikut kami makan siang?'",
        },
        {
          "id": "L15",
          "title": "Refleksi Jujur",
          "desc":
              "Beritahu seseorang 'Aku sangat menghargai ketika kamu melakukan X' dan jelaskan alasannya.",
        },
        {
          "id": "L16",
          "title": "Celah Keingintahuan",
          "desc":
              "Tanyakan kepada seseorang 'Aku selalu penasaran, bagaimana sebenarnya cara kerja X?'",
        },
        {
          "id": "L17",
          "title": "Memimpin Kelompok Kecil",
          "desc":
              "Ajukan pertanyaan yang membutuhkan jawaban dari 2 atau 3 orang dalam kelompok.",
        },
        {
          "id": "L18",
          "title": "Pujian Tulus",
          "desc":
              "Puji seseorang tentang sifat kepribadiannya (misalnya, 'Kamu adalah pendengar yang hebat').",
        },
        {
          "id": "L19",
          "title": "Pengalaman Bersama",
          "desc":
              "Ucapkan 'Aku juga pernah berada di situasi itu' selama percakapan.",
        },
        {
          "id": "L20",
          "title": "Jeda Bermakna",
          "desc":
              "Biarkan keheningan terjadi dalam percakapan tanpa terburu-buru mengisinya.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "Awal yang Berani",
          "desc":
              "Mulailah percakapan dengan seseorang yang tidak terlalu kamu kenal baik.",
        },
        {
          "id": "ST2",
          "title": "Berbagi Jujur",
          "desc":
              "Bagikan cerita pribadi singkat atau pendapat dalam lingkungan kelompok.",
        },
        {
          "id": "ST3",
          "title": "Perdebatan",
          "desc":
              "Nyatakan ketidaksetujuan secara sopan terhadap pendapat seseorang dan jelaskan alasannya.",
        },
        {
          "id": "ST4",
          "title": "Masuk ke Kelompok",
          "desc":
              "Bergabunglah dalam percakapan kelompok dan berikan kontribusi satu kalimat yang bernas.",
        },
        {
          "id": "ST5",
          "title": "Memimpin Topik",
          "desc": "Angkat topik percakapan baru dalam kelompok sosial.",
        },
        {
          "id": "ST6",
          "title": "Pertanyaan Publik",
          "desc":
              "Ajukan pertanyaan dalam rapat umum atau dalam lingkungan ruang kelas.",
        },
        {
          "id": "ST7",
          "title": "Permintaan Berani",
          "desc":
              "Tanyakan kepada orang asing apakah kamu boleh duduk di sebelah mereka di kafe atau taman.",
        },
        {
          "id": "ST8",
          "title": "Jembatan Percakapan",
          "desc":
              "Perkenalkan dua orang yang tidak saling mengenal dan temukan satu persamaan di antara mereka.",
        },
        {
          "id": "ST9",
          "title": "Kebutuhan Asertif",
          "desc":
              "Mintalah seseorang secara sopan untuk bergeser atau berhenti melakukan sesuatu yang mengganggumu.",
        },
        {
          "id": "ST10",
          "title": "Pendongeng",
          "desc":
              "Ambil peran utama dalam menceritakan sebuah kisah kepada kelompok yang terdiri dari 3 orang atau lebih.",
        },
        {
          "id": "ST11",
          "title": "Tantangan Terbuka",
          "desc":
              "Tantang pendapat umum dalam kelompok dengan cara yang ramah dan penuh hormat.",
        },
        {
          "id": "ST12",
          "title": "Inisiatif Sosial",
          "desc":
              "Jadilah orang pertama yang mengucapkan 'Halo' kepada semua orang saat memasuki ruangan.",
        },
        {
          "id": "ST13",
          "title": "Mendengar Empatis",
          "desc":
              "Dengarkan seseorang yang sedang meluapkan keluh kesahnya dan berikan respons yang mendukung.",
        },
        {
          "id": "ST14",
          "title": "Presentasi Publik",
          "desc":
              "Bicaralah selama 1-2 menit tentang topik yang kamu sukai dalam sebuah pertemuan sosial.",
        },
        {
          "id": "ST15",
          "title": "Berbagi Rentan",
          "desc":
              "Akui kepada kelompok bahwa kamu sempat gugup tentang sesuatu, lalu tertawalah bersama.",
        },
        {
          "id": "ST16",
          "title": "Menetapkan Batasan",
          "desc":
              "Tolak undangan yang tidak ingin kamu hadiri secara sopan tanpa memberikan penjelasan berlebihan.",
        },
        {
          "id": "ST17",
          "title": "Mediator Aktif",
          "desc":
              "Bantu dua orang menemukan jalan tengah dalam sebuah perselisihan.",
        },
        {
          "id": "ST18",
          "title": "Pujian Publik",
          "desc":
              "Puji usaha atau pencapaian seseorang secara terbuka dalam kelompok.",
        },
        {
          "id": "ST19",
          "title": "Pendekatan Langsung",
          "desc":
              "Mintalah bantuan atau saran yang kamu butuhkan secara langsung kepada seseorang.",
        },
        {
          "id": "ST20",
          "title": "Pengalihan Percakapan",
          "desc":
              "Ganti arah percakapan dengan mulus dari topik yang membosankan ke topik yang menarik.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "Hadiah",
          "desc":
              "Berikan hadiah kecil atau makanan kepada seseorang dan katakan 'Aku pikir kamu akan menyukai ini'.",
        },
        {
          "id": "B2",
          "title": "Memimpin Berani",
          "desc":
              "Berikan saran rencana atau tempat untuk dikunjungi kepada sekelompok kecil orang.",
        },
        {
          "id": "B3",
          "title": "Apresiasi",
          "desc":
              "Beritahu seseorang secara spesifik mengapa kamu menghargai kehadiran mereka di hidupmu.",
        },
        {
          "id": "B4",
          "title": "Tuan Rumah Sosial",
          "desc":
              "Organisasikan acara kumpul-kumpul kecil atau janji temu kopi untuk beberapa orang.",
        },
        {
          "id": "B5",
          "title": "Penyelaman Dalam",
          "desc":
              "Lakukan percakapan yang mendalam dan bermakna dengan seseorang selama lebih dari 15 minutes.",
        },
        {
          "id": "B6",
          "title": "Puncak Keyakinan",
          "desc":
              "Mulailah percakapan dengan seseorang yang menurutmu mengintimidasi.",
        },
        {
          "id": "B7",
          "title": "Apresiasi Publik",
          "desc":
              "Sampaikan ucapan selamat yang singkat dan positif atau pujian terbuka kepada seseorang dalam kelompok.",
        },
        {
          "id": "B8",
          "title": "Pembuat Batasan",
          "desc":
              "Katakan 'Tidak' pada sebuah permintaan dengan tegas tetapi ramah, tanpa penjelasan berlebihan.",
        },
        {
          "id": "B9",
          "title": "Permintaan Langsung",
          "desc":
              "Mintalah waktu mengobrol 10 menit atau bimbingan kepada seseorang yang kamu kagumi.",
        },
        {
          "id": "B10",
          "title": "Memimpin Emosional",
          "desc":
              "Mulailah percakapan tentang perasaan atau kesehatan mental bersama seorang teman.",
        },
        {
          "id": "B11",
          "title": "Mediator Sosial",
          "desc":
              "Bantu dua orang menyelesaikan konflik kecil melalui percakapan yang tenang.",
        },
        {
          "id": "B12",
          "title": "Pujian Berani",
          "desc":
              "Katakan kepada orang asing hal yang benar-benar kamu kagumi tentang dirinya.",
        },
        {
          "id": "B13",
          "title": "Langkah Jejaring",
          "desc":
              "Perkenalkan dirimu kepada seorang profesional di bidangmu dan mintalah saran.",
        },
        {
          "id": "B14",
          "title": "Kebenaran Berani",
          "desc":
              "Katakan kepada seseorang kebenaran yang sulit tetapi bermanfaat bagi hubungan kalian.",
        },
        {
          "id": "B15",
          "title": "Mekar Penuh",
          "desc":
              "Adakan acara sosial kecil dan pastikan setiap tamu merasa disambut dengan baik.",
        },
        {
          "id": "B16",
          "title": "Pembicara Publik",
          "desc":
              "Menjadi sukarelawan untuk berbicara atau memimpin sebagian kecil dari rapat atau acara.",
        },
        {
          "id": "B17",
          "title": "Memimpin Rentan",
          "desc":
              "Bagikan perjuangan yang telah kamu menangkan untuk menyemangati orang lain.",
        },
        {
          "id": "B18",
          "title": "Permintaan Maaf Berani",
          "desc":
              "Mulailah percakapan untuk meminta maaf atas kesalahan masa lalu, meskipun itu sudah lama terjadi.",
        },
        {
          "id": "B19",
          "title": "Mentor",
          "desc":
              "Tawarkan diri untuk membantu seseorang yang kurang berpengalaman darimu dalam suatu keahlian.",
        },
        {
          "id": "B20",
          "title": "Arsitek Sosial",
          "desc":
              "Buat tradisi sosial baru atau acara kumpul-kumpul berkala untuk sekelompok teman.",
        },
      ],
    },
    'ga': {
      "Seedling": [
        {
          "id": "S1",
          "title": "An Chéad Chéim",
          "desc":
              "Déan teagmháil súl agus meangadh gáire a dhéanamh ar dhuine amháin inniu.",
        },
        {
          "id": "S2",
          "title": "Dia Duit Simplí",
          "desc": "Abair 'Dia duit' nó 'Maidin mhaith' le comharsa.",
        },
        {
          "id": "S3",
          "title": "An Buíochas",
          "desc": "Abair 'Go raibh maith agat' go soiléir le siopadóir.",
        },
        {
          "id": "S4",
          "title": "An Breathnú",
          "desc":
              "Tabhair faoi deara rud éigin dearfach faoi dhuine strainséartha agus déan meangadh gáire.",
        },
        {
          "id": "S5",
          "title": "An Sméideadh Ciúin",
          "desc": "Sméid do lámh chuig duine a n-aithníonn tú i gcéin.",
        },
        {
          "id": "S6",
          "title": "An Doras a Choinneáil",
          "desc": "Coinnigh an doras oscailte do dhuine taobh thiar díot.",
        },
        {
          "id": "S7",
          "title": "An Nod",
          "desc":
              "Tabhair nod cairdiúil le do cheann do chomhghleacaí agus tú ag dul tharstu.",
        },
        {
          "id": "S8",
          "title": "An Scáthán",
          "desc":
              "Cleacht do 'mheangadh gáire muiníneach' sa scáthán ar feadh 1 nóiméad.",
        },
        {
          "id": "S9",
          "title": "An Sracfhéachaint",
          "desc":
              "Féach ar dhuine ar feadh 2 shoicind, ansin déan meangadh gáire agus breathnaigh ar shiúl.",
        },
        {
          "id": "S10",
          "title": "An Moladh Ciúin",
          "desc": "Scríobh trácht deas ar phostáil meán sóisialta duine éigin.",
        },
        {
          "id": "S11",
          "title": "Comhroinnt Spáis",
          "desc":
              "Suigh in aice le duine éigin i limistéar poiblí gan breathnú ar shiúl láithreach.",
        },
        {
          "id": "S12",
          "title": "An tAitheantas Simplí",
          "desc":
              "Abair 'Gabh mo leithscéal' go béasach nuair a théann tú thar dhuine i halla.",
        },
        {
          "id": "S13",
          "title": "An Beannacht Te",
          "desc": "Abair 'Dia duit' le tiománaí seachadta nó teachtaire.",
        },
        {
          "id": "S14",
          "title": "An Sméideadh Beag",
          "desc": "Sméid do lámh chuig leanbh nó peata (le cead an úinéara).",
        },
        {
          "id": "S15",
          "title": "An Meangadh Bog",
          "desc": "Déan meangadh gáire ar thriúr duine difriúil inniu.",
        },
        {
          "id": "S16",
          "title": "Dúshlán an Teagmhála Súl",
          "desc":
              "Coinnigh teagmháil súl le fuisिटीoir go dtí go mbreathnaíonn siad ar shiúl ar dtús.",
        },
        {
          "id": "S17",
          "title": "An Anáil Shéimh",
          "desc":
              "Tóg 3 anáil dhomhain sula dtéann tú isteach i spás sóisialta inniu.",
        },
        {
          "id": "S18",
          "title": "An Láithreacht",
          "desc":
              "Seas i limistéar plódaithe ar feadh 5 nóiméad gan féachaint ar do ghuthán.",
        },
        {
          "id": "S19",
          "title": "An Nod Ócáideach",
          "desc":
              "Nodaigh do cheann do dhuine strainséartha a dhéanann teagmháil súl leat.",
        },
        {
          "id": "S20",
          "title": "An Ghlór Bhog",
          "desc":
              "Abair 'Bíodh lá deas agat' le duine éigin agus tú ag fágáil siopa.",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "An Moladh",
          "desc":
              "Tabhair moladh ó chroí do chomhghleacaí nó do chomhscoláire.",
        },
        {
          "id": "SP2",
          "title": "An Cheist",
          "desc": "Iarr an t-am nó treoracha ar dhuine strainséartha.",
        },
        {
          "id": "SP3",
          "title": "Comhrá Beag",
          "desc":
              "Fiafraigh de dhuine éigin 'Conas atá do lá ag dul?' agus éist leis an bhfreagra.",
        },
        {
          "id": "SP4",
          "title": "An tAchainí",
          "desc": "Iarr cabhair ar fhostaí siopa chun mír shonrach a aimsiú.",
        },
        {
          "id": "SP5",
          "title": "An tOrdú",
          "desc":
              "Ordaigh deoch nó bia agus fiafraigh den fhoireann conas atá siad ag déanamh.",
        },
        {
          "id": "SP6",
          "title": "An Beannacht",
          "desc": "Tabhair tú féin in aithne do dhuine nua i do cheantar.",
        },
        {
          "id": "SP7",
          "title": "Caint na Haimsir",
          "desc": "Luaigh an aimsir le duine éigin agus tú ag fanacht i líne.",
        },
        {
          "id": "SP8",
          "title": "An Fhiosrúchán Simplí",
          "desc":
              "Fiafraigh de chomhghleacaí 'Cad a rinne tú ag an deireadh seachtaine?'",
        },
        {
          "id": "SP9",
          "title": "An Tairiscint Cabhrach",
          "desc":
              "Fiafraigh de dhuine éigin 'An bhfuil cabhair uait leis sin?' má fheictear go bhfuil siad ag streachailt.",
        },
        {
          "id": "SP10",
          "title": "An Tuairim",
          "desc":
              "Fiafraigh de chara 'Cad a cheapann tú faoi seo?' faoi rud beag.",
        },
        {
          "id": "SP11",
          "title": "An Deimhniú",
          "desc":
              "Deimhnigh mionsonra le duine strainséartha (e.g., 'An é seo an líne cheart?').",
        },
        {
          "id": "SP12",
          "title": "An Spás Comhroinnte",
          "desc":
              "Déan trácht beag faoin timpeallacht (e.g., 'Tá sé an-phlódaithe anseo').",
        },
        {
          "id": "SP13",
          "title": "An Fabhar Beag",
          "desc":
              "Iarr ar dhuine rud éigin a thabhairt duit (cosúil le naipcín) ag bord.",
        },
        {
          "id": "SP14",
          "title": "An tAiseolas Te",
          "desc":
              "Inis do fhreastalaí go raibh an bia go hiontach sula bhfágann tú.",
        },
        {
          "id": "SP15",
          "title": "An Seiceáil Ócáideach",
          "desc":
              "Seol téacs 'Conas atá tú?' chuig duine nár labhair tú leis le mí anuas.",
        },
        {
          "id": "SP16",
          "title": "An Cheist Oscailte",
          "desc":
              "Fiafraigh de dhuine éigin 'Cá bhfuil an áit is fearr leat chun cuairt a thabhairt uirthi sa chathair seo?'",
        },
        {
          "id": "SP17",
          "title": "An Riosca Is Lú",
          "desc":
              "Fiafraigh de dhuine strainséartha an bhfuil a fhios acu cá bhfuil an leithreas is gaire.",
        },
        {
          "id": "SP18",
          "title": "Moladh Míre",
          "desc":
              "Inis do dhuine éigin go dtaitníonn a mbróga/mála/oiriúint leat.",
        },
        {
          "id": "SP19",
          "title": "An Sos Béasach",
          "desc":
              "Feitheamh go mbeidh duine críochnaithe ag labhairt go hiomlán sula bhfreagraíonn tú iad.",
        },
        {
          "id": "SP20",
          "title": "An Sméideadh Cairdiúil",
          "desc":
              "Sméid do lámh agus abair 'Slán' le duine éigin a raibh idirghníomhaíocht ghearr agat leo díreach anois.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "Lorgaire Tuairimí",
          "desc":
              "Iarr tuairim ar dhuine éigin faoi leabhar, scannán, nó amhrán.",
        },
        {
          "id": "L2",
          "title": "An Mionsonra",
          "desc":
              "Cuir ceist leantach tar éis do dhuine éigin rud éigin a insint duit fúthu féin.",
        },
        {
          "id": "L3",
          "title": "An Moladh",
          "desc":
              "Iarr moladh ar dhuine strainséartha maidir le háit mhaith le hithe in aice láimhe.",
        },
        {
          "id": "L4",
          "title": "An Ceangal",
          "desc":
              "Aimsigh spéis choiteann le duine éigin agus labhair faoi ar feadh 2 nóiméad.",
        },
        {
          "id": "L5",
          "title": "An Lámh Chúnta",
          "desc":
              "Tairg cabhair a thabhairt do dhuine éigin le tasc beag (cosúil le mála a iompar).",
        },
        {
          "id": "L6",
          "title": "An Breathnú Sóisialta",
          "desc":
              "Tosaigh comhrá bunaithe ar rud éigin atá ag tarlú thart timpeall oraibh beirt.",
        },
        {
          "id": "L7",
          "title": "An Cheist Oscailte",
          "desc":
              "Fiafraigh de dhuine éigin 'Conas a thosaigh tú sa chineál seo oibre?'",
        },
        {
          "id": "L8",
          "title": "An tÉisteoir Gníomhach",
          "desc":
              "Éist le duine éigin ar feadh 3 nóiméad gan cur isteach orthu, ansin déan achoimre ar cad a dúirt siad.",
        },
        {
          "id": "L9",
          "title": "An Gáire Comhroinnte",
          "desc": "Inis scéal gearr greannmhar nó magadh do ghrúpa beag.",
        },
        {
          "id": "L10",
          "title": "An Fhiosracht",
          "desc":
              "Fiafraigh de dhuine éigin as cá bhfuil siad agus cad a thaitníonn leo faoin áit sin.",
        },
        {
          "id": "L11",
          "title": "An tSuim Ó chroí",
          "desc":
              "Fiafraigh de chomhghleacaí faoina gcuid caitheamh aimsire lasmuigh den obair.",
        },
        {
          "id": "L12",
          "title": "An Chomhairle Bhog",
          "desc":
              "Tabhair leid chúnta do dhuine éigin ar rud éigin a bhfuil tú go maith aige.",
        },
        {
          "id": "L13",
          "title": "Nod an Ghrúpa",
          "desc": "Aontaigh le pointe duine éigin i bplé grúpa bhig.",
        },
        {
          "id": "L14",
          "title": "An Tabhairt cuireadh Ócáideach",
          "desc":
              "Fiafraigh de dhuine éigin 'Ar mhaith leat bualadh isteach linn le haghaidh lóin?'",
        },
        {
          "id": "L15",
          "title": "An Machnamh Macánta",
          "desc":
              "Inis do dhuine éigin 'Bhí an-bhuíochas agam as nuair a rinne tú X' agus mínigh cén fáth.",
        },
        {
          "id": "L16",
          "title": "Bearna na Fiosrachta",
          "desc":
              "Fiafraigh de dhuine éigin 'Bhí mé ag smaoineamh i gcónaí, conas a oibríonn X i ndáiríre?'",
        },
        {
          "id": "L17",
          "title": "Treoir an Ghrúpa Bhig",
          "desc":
              "Cuir ceist a éilíonn ar 2 nó 3 dhuine i ngrúpa freagra a thabhairt.",
        },
        {
          "id": "L18",
          "title": "An Moladh Fíor",
          "desc":
              "Mol duine éigin ar thréith phearsantachta (e.g., 'Is éisteoir iontach tú').",
        },
        {
          "id": "L19",
          "title": "An Taithí Chomhroinnte",
          "desc": "Abair 'Bhí mé sa chás sin freisin' le linn comhrá.",
        },
        {
          "id": "L20",
          "title": "An Sos Bríoch",
          "desc":
              "Lig do thost tarlú i gcomhrá gan a bheith ag rith chun é a líonadh.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "An Tús Cróga",
          "desc":
              "Tosaigh comhrá le duine nach bhfuil aithne mhaith agat orthu.",
        },
        {
          "id": "ST2",
          "title": "An Comhroinnt Macánta",
          "desc": "Comhroinn scéal beag pearsanta nó tuairim i suíomh grúpa.",
        },
        {
          "id": "ST3",
          "title": "An Díospóireacht",
          "desc":
              "Easaontaigh go béasach le tuairim duine éigin agus mínigh cén fáth.",
        },
        {
          "id": "ST4",
          "title": "Iontráil Ghrúpa",
          "desc":
              "Bí i dteannta comhrá grúpa agus cuir abairt mhachnamhach leis.",
        },
        {
          "id": "ST5",
          "title": "Treoir an Ábhair",
          "desc": "Tabhair ábhar comhrá nua suas i ngrúpa sóisialta.",
        },
        {
          "id": "ST6",
          "title": "An Cheist Phoiblí",
          "desc": "Cuir ceist i gcruinniú poiblí nó i suíomh seomra ranga.",
        },
        {
          "id": "ST7",
          "title": "An tAchainí Dhána",
          "desc":
              "Fiafraigh de dhuine strainséartha an féidir leat suí in aice leo ag caifé nó i bparc.",
        },
        {
          "id": "ST8",
          "title": "Droichead Comhrá",
          "desc":
              "Tabhair beirt nach bhfuil aithne acu ar a chéile in aithne dá chéile agus aimsigh rud éigin coiteann.",
        },
        {
          "id": "ST9",
          "title": "An Gá Dearfa",
          "desc":
              "Iarr go béasach ar dhuine éigin bogadh nó stop a chur le rud éigin a chuireann as duit.",
        },
        {
          "id": "ST10",
          "title": "An Scéalaí",
          "desc":
              "Tóg an treoir maidir le scéal a insint do ghrúpa de 3 dhuine nó níos mó.",
        },
        {
          "id": "ST11",
          "title": "An Dúshlán Oscailte",
          "desc":
              "Tabhair dúshlán do thuairim choiteann i ngrúpa ar bhealach cairdiúil, measúil.",
        },
        {
          "id": "ST12",
          "title": "An Tionscnamh Sóisialta",
          "desc":
              "Bí ar an gcéad duine a deir 'Dia daoibh' le gach duine agus tú ag dul isteach i seomra.",
        },
        {
          "id": "ST13",
          "title": "An tÉisteacht Comhbhách",
          "desc":
              "Éist le duine éigin ag cur a gcroí amach agus tabhair freagra tacúil.",
        },
        {
          "id": "ST14",
          "title": "An Cur i Láthair Poiblí",
          "desc":
              "Labhair ar feadh 1-2 nóiméad faoi ábhar is breá leat i gcruinniú sóisialta.",
        },
        {
          "id": "ST15",
          "title": "An Comhroinnt Leochaileach",
          "desc":
              "Admhaigh do ghrúpa go raibh tú neirbhíseach faoi rud éigin, edus déan gáire faoi le chéile.",
        },
        {
          "id": "ST16",
          "title": "An Socrú Teorann",
          "desc":
              "Diúltaigh go béasach do chuireadh nach dteastaíonn uait freastal air gan an iomarca mínithe a thabhairt.",
        },
        {
          "id": "ST17",
          "title": "An tIdirghabhálaí Gníomhach",
          "desc": "Cabhraigh le beirt comhréiteach a aimsiú in easaontas.",
        },
        {
          "id": "ST18",
          "title": "An Moladh Poiblí",
          "desc":
              "Mol iarracht nó gnóthachtáil duine éigin go poiblí i ngrúpa.",
        },
        {
          "id": "ST19",
          "title": "An Cur Chuige Díreach",
          "desc":
              "Iarr go díreach ar dhuine fabhar nó píosa comhairle atá uait.",
        },
        {
          "id": "ST20",
          "title": "Pivot an Chomhrá",
          "desc":
              "Aistrigh comhrá go réidh ó ábhar leadránach go hábhar suimiúil.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "An Bronntanas",
          "desc":
              "Tabhair féirín beag do dhuine éigin agus abair 'Cheap mé go dtaitneodh sé seo leat'.",
        },
        {
          "id": "B2",
          "title": "An Treoir Dhána",
          "desc":
              "Mol plean nó áit chun cuairt a thabhairt air do ghrúpa beag daoine.",
        },
        {
          "id": "B3",
          "title": "An Léiriú Buíochais",
          "desc":
              "Inis do dhuine go sonrach cén fáth a bhfuil tú buíoch as iad a bheith i do shaol.",
        },
        {
          "id": "B4",
          "title": "An tÓstach Sóisialta",
          "desc": "Eagraigh cruinniú beag nó coinne caife do chúpla duine.",
        },
        {
          "id": "B5",
          "title": "An Tumadh Domhain",
          "desc":
              "Bíodh comhrá domhain, bríoch agat le duine éigin ar feadh níos mó ná 15 nóiméad.",
        },
        {
          "id": "B6",
          "title": "Buaic na Muiníne",
          "desc":
              "Tosaigh comhrá le duine éigin a bhraitheann tú atá imeaglach.",
        },
        {
          "id": "B7",
          "title": "An Tósta Poiblí",
          "desc":
              "Déan tósta gearr, dearfach nó moladh poiblí do dhuine éigin i ngrúpa.",
        },
        {
          "id": "B8",
          "title": "An Socróir Teorann",
          "desc":
              "Abair 'Níl' le hiarratas go daingean ach go cineálta, gan an iomarca mínithe a thabhairt.",
        },
        {
          "id": "B9",
          "title": "An tAchainí Dhíreach",
          "desc":
              "Iarr comhrá 10 nóiméad nó meantóireacht ar dhuine a bhfuil meas agat orthu.",
        },
        {
          "id": "B10",
          "title": "An Treoir Mhothúchánach",
          "desc":
              "Tosaigh comhrá faoi mhothúcháin nó faoi mheabhairshláinte le cara.",
        },
        {
          "id": "B11",
          "title": "An tIdirghabhálaí Sóisialta",
          "desc":
              "Cabhraigh le beirt achrann beag a réiteach trí chomhrá socair.",
        },
        {
          "id": "B12",
          "title": "An Moladh Dána",
          "desc":
              "Inis do dhuine nach bhfuil aithne agat orthu i ndáiríre faoi rud éigin a bhfuil meas agat orthu go fírinneach.",
        },
        {
          "id": "B13",
          "title": "An tIompar Líonraithe",
          "desc":
              "Tabhair tú féin in aithne do ghairmí i do réimse agus iarr comhairle orthu.",
        },
        {
          "id": "B14",
          "title": "An Fhírinne Chróga",
          "desc":
              "Inis fírinne do dhuine éigin atá deacair ach atá cabhrach don ghaolmhaireacht.",
        },
        {
          "id": "B15",
          "title": "An Lánbhláth",
          "desc":
              "Eagraigh imeacht sóisialta beag agus déan cinnte go mbraitheann gach aoi go bhfuil fáilte rompu.",
        },
        {
          "id": "B16",
          "title": "An Cainteoir Poiblí",
          "desc":
              "Voluntáil chun labhairt nó chun cuid bheag de chruinniú nó d'imeacht a threorú.",
        },
        {
          "id": "B17",
          "title": "An Treoir Leochaileach",
          "desc":
              "Comhroinn streachailt a d'éirigh leat a shárú chun duine éigin eile a spreagadh.",
        },
        {
          "id": "B18",
          "title": "An Leithscéal Dána",
          "desc":
              "Tosaigh comhrá chun leithscéal a ghabháil as botún san am atá thart, fiú más fada ó shin a tharla sé.",
        },
        {
          "id": "B19",
          "title": "An Meantóir",
          "desc":
              "Tairg cabhair a thabhairt do dhuine éigin a bhfuil níos lú taithí acu ná tú le scil ar leith.",
        },
        {
          "id": "B20",
          "title": "An tAiltire Sóisialta",
          "desc":
              "Cruthaigh traidisiún sóisialta nua nó cruinniú athfhillteach do ghrúpa cairde.",
        },
      ],
    },
    'sv': {
      "Seedling": [
        {
          "id": "S1",
          "title": "Det första steget",
          "desc": "Etablera ögonkontakt och le mot en person idag.",
        },
        {
          "id": "S2",
          "title": "Ett enkelt hej",
          "desc": "Säg 'God morgon' eller 'Hej' till en granne.",
        },
        {
          "id": "S3",
          "title": "Tacket",
          "desc": "Säg 'Tack' tydligt till en butiksanställd.",
        },
        {
          "id": "S4",
          "title": "Observationen",
          "desc": "Lägg märke till något positivt hos en främling och le.",
        },
        {
          "id": "S5",
          "title": "Den tysta vinkningen",
          "desc": "Vinka till någon du känner igen på avstånd.",
        },
        {
          "id": "S6",
          "title": "Håll dörren",
          "desc": "Håll dörren öppen för någon bakom dig.",
        },
        {
          "id": "S7",
          "title": "Nickningen",
          "desc": "Ge en vänlig nick till en kollega när du går förbi.",
        },
        {
          "id": "S8",
          "title": "Spegeln",
          "desc": "Öva på ditt 'självsäkra leende' i spegeln i 1 minut.",
        },
        {
          "id": "S9",
          "title": "Den korta blicken",
          "desc": "Titta på någon i 2 sekunder, le och se bort.",
        },
        {
          "id": "S10",
          "title": "Den tysta berömmen",
          "desc": "Skriv en trevlig kommentar på sociala medier till någon.",
        },
        {
          "id": "S11",
          "title": "Dela utrymmet",
          "desc":
              "Sätt dig bredvid någon på en offentlig plats utan att se bort på en gång.",
        },
        {
          "id": "S12",
          "title": "Enkel höflighet",
          "desc":
              "Säg 'Ursäkta mig' på ett höfligt sätt när du går förbi någon i en korridor.",
        },
        {
          "id": "S13",
          "title": "Den varma hälsningen",
          "desc": "Säg 'Hej' till ett bud eller en chaufför.",
        },
        {
          "id": "S14",
          "title": "Den lilla vinkningen",
          "desc":
              "Vinka till ett barn eller ett husdjur (med ägarens tillstånd).",
        },
        {
          "id": "S15",
          "title": "Det mjuka leendet",
          "desc": "Le mot tre olika personer idag.",
        },
        {
          "id": "S16",
          "title": "Ögonkontaktsutmaningen",
          "desc": "Håll ögonkontakt med en kassörska tills de ser bort först.",
        },
        {
          "id": "S17",
          "title": "Det lugna andetaget",
          "desc":
              "Ta 3 djupa andetag innan du går in i ett socialt utrymme idag.",
        },
        {
          "id": "S18",
          "title": "Närvaron",
          "desc":
              "Stå i ett befolkat område i 5 minuter utan att titta på din telefon.",
        },
        {
          "id": "S19",
          "title": "Den informella nickningen",
          "desc": "Nicka till en främling som etablerar ögonkontakt med dig.",
        },
        {
          "id": "S20",
          "title": "Den milda rösten",
          "desc": "Säg 'Ha en bra dag' till någon när du lämnar en butik.",
        },
      ],
      "Sprout": [
        {
          "id": "SP1",
          "title": "Komplimangen",
          "desc":
              "Ge en uppriktig komplimang till en kollega eller medstudent.",
        },
        {
          "id": "SP2",
          "title": "Frågan",
          "desc": "Fråga en främling om klockan eller vägbeskrivning.",
        },
        {
          "id": "SP3",
          "title": "Småprat",
          "desc": "Fråga någon 'Hur går din dag?' och lyssna på svaret.",
        },
        {
          "id": "SP4",
          "title": "Förfrågan",
          "desc":
              "Fråga en butiksanställd om hjälp med att hitta en specifik vara.",
        },
        {
          "id": "SP5",
          "title": "Beställningen",
          "desc": "Beställ en dryck eller mat och fråga personalen hur de mår.",
        },
        {
          "id": "SP6",
          "title": "Hälsningen",
          "desc": "Introducera dig själv för någon ny i ditt närområde.",
        },
        {
          "id": "SP7",
          "title": "Väderpratet",
          "desc": "Nämn vädret för någon medan du väntar i en kö.",
        },
        {
          "id": "SP8",
          "title": "Den enkla frågan",
          "desc": "Fråga en kollega 'Vad gjorde du i helgen?'",
        },
        {
          "id": "SP9",
          "title": "Hjälperbjudandet",
          "desc":
              "Fråga någon 'Behöver du hjälp med det där?' om de ser ut att kämpa.",
        },
        {
          "id": "SP10",
          "title": "Meningen",
          "desc":
              "Fråga en vän 'Vad tycker du om den här?' om ett litet föremål.",
        },
        {
          "id": "SP11",
          "title": "Bekräftelsen",
          "desc":
              "Bekräfta en detalj med en främling (t.ex. 'Är det här den rätta kön?').",
        },
        {
          "id": "SP12",
          "title": "Det delade utrymmet",
          "desc":
              "Kom med en liten kommentar om omgivningen (t.ex. 'Det är väldigt mycket folk här').",
        },
        {
          "id": "SP13",
          "title": "Den lilla tjänsten",
          "desc":
              "Fråga någon om att skicka dig något (som en servett) vid bordet.",
        },
        {
          "id": "SP14",
          "title": "Den varma feedbacken",
          "desc":
              "Berätta för en servitör att maten var fantastisk innan du går.",
        },
        {
          "id": "SP15",
          "title": "Informell uppföljning",
          "desc":
              "Skicka ett 'Hur är läget?'-textmeddelande till någon du inte har pratat med på en månad.",
        },
        {
          "id": "SP16",
          "title": "Den öppna frågan",
          "desc":
              "Fråga någon 'Vilket är ditt favoritställe att besöka i den här staden?'",
        },
        {
          "id": "SP17",
          "title": "Den minsta risken",
          "desc": "Fråga en främling om de vet var närmaste toalett ligger.",
        },
        {
          "id": "SP18",
          "title": "Föremålsros",
          "desc": "Berätta för någon att du gillar deras skor/väska/tillbehör.",
        },
        {
          "id": "SP19",
          "title": "Den hövliga pausen",
          "desc":
              "Vänta tills någon har pratat helt klart innan du svarar dem.",
        },
        {
          "id": "SP20",
          "title": "Det vänliga avskedet",
          "desc":
              "Vinka och säg 'Hej då' till någon du just hade en kort interaktion med.",
        },
      ],
      "Leaf": [
        {
          "id": "L1",
          "title": "Meningstagaren",
          "desc": "Fråga någon om deras mening om en bok, film eller sång.",
        },
        {
          "id": "L2",
          "title": "Detaljen",
          "desc":
              "Ställ en uppföljningsfråga efter att någon har berättat något om sig själv för dig.",
        },
        {
          "id": "L3",
          "title": "Rekommendationen",
          "desc":
              "Fråga en främling om en rekommendation om ett bra ställe att äta på i närheten.",
        },
        {
          "id": "L4",
          "title": "Kopplingen",
          "desc":
              "Hitta ett gemensamt intresse med någon och prata om det i 2 minuter.",
        },
        {
          "id": "L5",
          "title": "Den hjälpande handen",
          "desc":
              "Erbjud dig att hjälpa någon med en liten uppgift (som att bära en kasse).",
        },
        {
          "id": "L6",
          "title": "Social observation",
          "desc":
              "Starta en konversation baserad på något som händer runt er båda.",
        },
        {
          "id": "L7",
          "title": "Den öppna frågan",
          "desc": "Fråga någon 'Hur hamnade du i det här yrket?'",
        },
        {
          "id": "L8",
          "title": "Den aktiva lyssnaren",
          "desc":
              "Lyssna på någon i 3 minuter utan att avbryta, och sammanfatta sedan vad de sa.",
        },
        {
          "id": "L9",
          "title": "Det delade skrattet",
          "desc":
              "Berätta en kort, rolig historia eller ett skämt för en liten grupp.",
        },
        {
          "id": "L10",
          "title": "Nykfikenheten",
          "desc":
              "Fråga någon var de är ifrån och vad de gillar med det stället.",
        },
        {
          "id": "L11",
          "title": "Den uppriktiga intresset",
          "desc": "Fråga en kollega om deras hobbyer utanför jobbet.",
        },
        {
          "id": "L12",
          "title": "Det milda rådet",
          "desc": "Ge någon ett användbart tips om något du är bra på.",
        },
        {
          "id": "L13",
          "title": "Gruppnickningen",
          "desc": "Håll med om någons poäng i en liten gruppdiskussion.",
        },
        {
          "id": "L14",
          "title": "Den informella inbjudan",
          "desc": "Fråga någon 'Har du lust att hänga med oss på lunch?'",
        },
        {
          "id": "L15",
          "title": "Den ärliga reflektionen",
          "desc":
              "Berätta för någon 'Jag uppskattade verkligen när du gjorde X' och förklara varför.",
        },
        {
          "id": "L16",
          "title": "Nyfikenhetsgapet",
          "desc":
              "Fråga någon 'Jag har alltid undrat, hur fungerar egentligen X?'",
        },
        {
          "id": "L17",
          "title": "Mindre gruppledare",
          "desc":
              "Ställ ett fråga som kräver att 2 eller 3 personer i en grupp svarar.",
        },
        {
          "id": "L18",
          "title": "Det äkta komplimangen",
          "desc":
              "Ge någon en komplimang för ett personlighetsdrag (f.ex. 'Du är en god lyssnare').",
        },
        {
          "id": "L19",
          "title": "Den delade upplevelsen",
          "desc":
              "Säg 'Jag har varit i den situationen själv' under en konversation.",
        },
        {
          "id": "L20",
          "title": "Den meningsfulla pausen",
          "desc":
              "Tillåt att en tystnad uppstår i en konversation utan att skynda dig att fylla den.",
        },
      ],
      "Stem": [
        {
          "id": "ST1",
          "title": "Den modiga starten",
          "desc": "Starta en konversation med någon du inte känner så väl.",
        },
        {
          "id": "ST2",
          "title": "Det ärliga bidraget",
          "desc":
              "Dela en liten personlig historia eller åsikt i en gruppkontext.",
        },
        {
          "id": "ST3",
          "title": "Debatten",
          "desc": "Säg dig hövligt oense med någons åsikt och förklara varför.",
        },
        {
          "id": "ST4",
          "title": "Gruppingången",
          "desc":
              "Gå med i en gruppkonversation och bidra med en genomtänkt mening.",
        },
        {
          "id": "ST5",
          "title": "Temaledaren",
          "desc": "Introducera ett nytt samtaletema i en social grupp.",
        },
        {
          "id": "ST6",
          "title": "Den offentliga frågan",
          "desc":
              "Ställ en fråga i ett offentligt möte eller i en klassrumsmiljö.",
        },
        {
          "id": "ST7",
          "title": "Den dristiga förfrågan",
          "desc":
              "Fråga en främling om du kan sitta bredvid dem på ett café eller i en park.",
        },
        {
          "id": "ST8",
          "title": "Samtalsbron",
          "desc":
              "Introducera två personer som inte känner varandra och hitta en gemensam nämnare.",
        },
        {
          "id": "ST9",
          "title": "Det assertiva behovet",
          "desc":
              "Be någon hövligt att flytta på sig eller sluta göra något som stör dig.",
        },
        {
          "id": "ST10",
          "title": "Berättaren",
          "desc":
              "Ta ledningen i att berätta een historia för en grupp på 3 eller fler personer.",
        },
        {
          "id": "ST11",
          "title": "Den öppna utmaningen",
          "desc":
              "Utmana en vanlig åsikt i en grupp på ett vänligt och respektfullt sätt.",
        },
        {
          "id": "ST12",
          "title": "Socialt initiativ",
          "desc":
              "Vara den första personen att säga 'Hej' till alla när du går in i ett rum.",
        },
        {
          "id": "ST13",
          "title": "Den empatiska lyssningen",
          "desc":
              "Lyssna på någon som tömmer sitt hjärta, och ge en stödjande respons.",
        },
        {
          "id": "ST14",
          "title": "Den offentliga presentationen",
          "desc":
              "Prata i 1-2 minuter om ett tema du älskar i en social sammankomst.",
        },
        {
          "id": "ST15",
          "title": "Det sårbara bidraget",
          "desc":
              "Erkänn för en grupp att du var nervös för något, och le åt det tillsammans.",
        },
        {
          "id": "ST16",
          "title": "Sätta gränser",
          "desc":
              "Avböj hövligt en inbjudan du inte önskar delta på utan att överförklara.",
        },
        {
          "id": "ST17",
          "title": "Den aktiva medlaren",
          "desc": "Hjälp två personer att hitta en medelväg i en oenighet.",
        },
        {
          "id": "ST18",
          "title": "Det offentliga komplimangen",
          "desc": "Rosa offentligt någons insats eller prestation i en grupp.",
        },
        {
          "id": "ST19",
          "title": "Den direkta tillvägagångssättet",
          "desc": "Fråga någon direkt om en tjänst eller ett råd du behöver.",
        },
        {
          "id": "ST20",
          "title": "Samtalsvändningen",
          "desc":
              "Få en konversation att gå smidigt över från ett tråkigt tema till ett intressant.",
        },
      ],
      "Bloom": [
        {
          "id": "B1",
          "title": "Gåvan",
          "desc":
              "Ge en liten uppmärksamhet till någon och säg 'Jag tänkte du skulle gilla den här'.",
        },
        {
          "id": "B2",
          "title": "Den modiga ledningen",
          "desc":
              "Föreslå en plan eller en plats att besöka för en liten grupp människor.",
        },
        {
          "id": "B3",
          "title": "Uppskattningen",
          "desc":
              "Berätta för någon specifikt varför du uppskattar att ha dem i ditt liv.",
        },
        {
          "id": "B4",
          "title": "Den sociala värden",
          "desc":
              "Organisera en liten sammankomst eller en kaffedejt för några personer.",
        },
        {
          "id": "B5",
          "title": "Djupdykningen",
          "desc":
              "Ha en djup, meningsfull konversation med någon i över 15 minuter.",
        },
        {
          "id": "B6",
          "title": "Självförtroendetoppen",
          "desc": "Starta en konversation med någon du tycker är skrämmande.",
        },
        {
          "id": "B7",
          "title": "Den offentliga skålen",
          "desc":
              "Håll en kort, positiv skål eller ge ett erkännande till någon i en grupp.",
        },
        {
          "id": "B8",
          "title": "Grensesättaren",
          "desc":
              "Säg 'Nej' till en förfrågan på ett bestämt men vänligt sätt, utan att överförklara.",
        },
        {
          "id": "B9",
          "title": "Den direkta förfrågan",
          "desc":
              "Fråga någon du beundrar om en 10-minuters pratstund eller mentorskap.",
        },
        {
          "id": "B10",
          "title": "Den känslomässiga ledningen",
          "desc":
              "Starta en konversation om känslor eller mental hälsa med en vän.",
        },
        {
          "id": "B11",
          "title": "Social medlare",
          "desc":
              "Hjälp två personer att lösa en liten konflikt genom en lugn konversation.",
        },
        {
          "id": "B12",
          "title": "Det modiga komplimangen",
          "desc":
              "Berätta för en helt främmande person något du uppriktigt beundrar hos dem.",
        },
        {
          "id": "B13",
          "title": "Nätverkssteget",
          "desc":
              "Introducera dig själv för en fackperson inom ditt fält och fråga om råd.",
        },
        {
          "id": "B14",
          "title": "Den modiga sanningen",
          "desc":
              "Berätta för någon en sanning som är svår, men nyttig för förhållandet.",
        },
        {
          "id": "B15",
          "title": "Full blom",
          "desc":
              "Arrangera ett litet socialt arrangemang och se till att varje gäst känner sig välkommen.",
        },
        {
          "id": "B16",
          "title": "Den offentliga talaren",
          "desc":
              "Anmäl dig frivilligt till att prata eller leda en liten del av ett möte eller arrangemang.",
        },
        {
          "id": "B17",
          "title": "Sårbar ledning",
          "desc":
              "Dela en utmaning du har övervunnit för att uppmuntra någon annan.",
        },
        {
          "id": "B18",
          "title": "Den modiga ursäkten",
          "desc":
              "Starta en konversation för att säga förlåt för ett tidigare fel, även om det var länge sedan.",
        },
        {
          "id": "B19",
          "title": "Mentorn",
          "desc":
              "Erbjud dig att hjälpa någon som är mindre erfaren än dig med en färdighet.",
        },
        {
          "id": "B20",
          "title": "Den sociala arkitekten",
          "desc":
              "Skapa en ny social tradition eller en återkommande träff för ett kompisgäng.",
        },
      ],
    },
  };

  static List<Map<String, String>> getTasks(String level, String lang) =>
      translations[lang]?[level] ?? translations['en']![level]!;
  // NEW: Find a task by its ID across all levels
  static Map<String, String>? getTaskById(String id, String lang) {
    final levelData = translations[lang] ?? translations['en']!;
    for (var level in levelData.values) {
      for (var task in level) {
        if (task['id'] == id) return task;
      }
    }
    return null; // Return null if not found
  }
}

// --- SERVICES ---
class AuthService {
  final FirebaseAuth _auth = FirebaseAuth.instance;
  Stream<User?> get userStream => _auth.authStateChanges();
  Future<void> signOut() async => await _auth.signOut();
  Future<UserCredential?> signInWithEmail(
    String email,
    String password,
  ) async =>
      await _auth.signInWithEmailAndPassword(email: email, password: password);
  Future<UserCredential?> signUpWithEmail(
    String email,
    String password,
  ) async => await _auth.createUserWithEmailAndPassword(
    email: email,
    password: password,
  );
  Future<UserCredential?> signInWithGoogle() async {
    final GoogleSignInAccount? googleUser = await GoogleSignIn().signIn();
    if (googleUser == null) return null;
    final GoogleSignInAuthentication googleAuth =
        await googleUser.authentication;
    final AuthCredential credential = GoogleAuthProvider.credential(
      accessToken: googleAuth.accessToken,
      idToken: googleAuth.idToken,
    );
    return await _auth.signInWithCredential(credential);
  }

  Future<UserCredential?> signInAnonymously() async {
    // First, check if we have a local profile already saved
    LocalStorageService localStore = LocalStorageService();
    await localStore.init();
    Map<String, dynamic> localProfile = localStore.loadProfile();

    if (localProfile.isNotEmpty) {
      // The user has local data! We proceed with anonymous login,
      // but we will merge the local data back into the new account in the Wrapper.
      return await _auth.signInAnonymously();
    } else {
      return await _auth.signInAnonymously();
    }
  }
}

// --- USER SERVICE (Corrected and Verified) ---
class UserService {
  final FirebaseFirestore _db = FirebaseFirestore.instance;

  Stream<DocumentSnapshot> getUserStream(User user) {
    return _db.collection('users').doc(user.uid).snapshots();
  }

  Future<void> setupUserProfile(User user) async {
    DocumentReference userRef = _db.collection('users').doc(user.uid);
    DocumentSnapshot snap = await userRef.get();

    if (!snap.exists) {
      // If brand new user, create the document
      await userRef.set({
        'userName': user.displayName ?? "Brave Soul",
        'confidenceScore': 0.0,
        'totalPoints': 0,
        'currentStreak': 0,
        'bestStreak': 0,
        'streakFreezes': 0,
        'activeFreezes': 0,
        'completedTasks': [],
        'lastCompletedDate': DateTime.now().toIso8601String().split('T')[0],
        'createdAt': FieldValue.serverTimestamp(),
      });
    } else {
      // If existing user, ensure all NEW fields exist
      Map<String, dynamic> data = snap.data() as Map<String, dynamic>;
      Map<String, dynamic> updates = {};

      if (!data.containsKey('totalPoints')) updates['totalPoints'] = 0;
      if (!data.containsKey('streakFreezes')) updates['streakFreezes'] = 0;
      if (!data.containsKey('activeFreezes')) updates['activeFreezes'] = 0;
      if (!data.containsKey('lastCompletedDate'))
        updates['lastCompletedDate'] = DateTime.now().toIso8601String().split(
          'T',
        )[0];

      if (updates.isNotEmpty) {
        await userRef.update(updates);
      }
    }
  }

  // --- THE DECAY LOGIC ---
  // REPLACE THIS ENTIRE METHOD:
  Future<void> handleDailyCheckIn(User user) async {
    DocumentReference userRef = _db.collection('users').doc(user.uid);
    DocumentSnapshot snap = await userRef.get();

    if (!snap.exists) return;

    Map<String, dynamic> data = snap.data() as Map<String, dynamic>;
    final todayStr = DateFormat('yyyy-MM-dd').format(DateTime.now());
    final lastDateStr = data['lastCompletedDate'] as String? ?? '';

    if (lastDateStr.compareTo(todayStr) >= 0) return;

    final lastDate =
        DateTime.tryParse(lastDateStr) ??
        DateTime.now().subtract(const Duration(days: 100));
    final diff = DateTime.now().difference(lastDate).inDays;

    if (diff > 1) {
      int activeFreezes = data['activeFreezes'] as int? ?? 0;
      int currentStreak = data['currentStreak'] as int? ?? 0;
      double score = (data['confidenceScore'] as num? ?? 0).toDouble();

      if (activeFreezes > 0) {
        await userRef.update({'activeFreezes': FieldValue.increment(-1)});
      } else {
        double newScore = (score - 2.0).clamp(0.0, 100.0);
        await userRef.update({'currentStreak': 0, 'confidenceScore': newScore});
      }

      // Sync Local after Cloud Decay
      var local = GlobalSettings.userProfile.value;
      local = StreakEngine.processDailyUpdate(local);
      await GlobalSettings._storage.saveProfile(local);
      GlobalSettings.userProfile.value = local;
    }
  }

  Future<Map<String, dynamic>> getUserData(User user) async {
    DocumentSnapshot snap = await _db.collection('users').doc(user.uid).get();
    // FIX: Return empty map if missing, prevents crash in Profile/Progress/Shop screens
    if (!snap.exists) return {};
    return snap.data() as Map<String, dynamic>;
  }

  Future<void> completeTask(User user, String taskId, double points) async {
    DocumentReference userRef = _db.collection('users').doc(user.uid);

    await FirebaseFirestore.instance.runTransaction((transaction) async {
      DocumentSnapshot snap = await transaction.get(userRef);

      if (!snap.exists) throw Exception("User profile missing");

      Map<String, dynamic> data = snap.data() as Map<String, dynamic>;
      List completed = List.from(data['completedTasks'] ?? []);

      if (completed.contains(taskId)) return;

      completed.add(taskId);
      final todayStr = DateFormat('yyyy-MM-dd').format(DateTime.now());
      final lastDateStr = data['lastCompletedDate'] as String? ?? '';

      int currentStreak = data['currentStreak'] as int? ?? 0;
      int bestStreak = data['bestStreak'] as int? ?? 0;
      double newScore =
          (data['confidenceScore'] as num? ?? 0).toDouble() + points;
      int newPoints = (data['totalPoints'] as int? ?? 0) + points.round();

      // STREAK LOGIC
      if (lastDateStr == todayStr) {
        // Same day, unchanged
      } else if (lastDateStr ==
          DateFormat(
            'yyyy-MM-dd',
          ).format(DateTime.now().subtract(const Duration(days: 1)))) {
        currentStreak += 1;
        bestStreak = max(bestStreak, currentStreak);
      } else {
        currentStreak = 1;
        bestStreak = max(bestStreak, currentStreak);
      }

      transaction.update(userRef, {
        'confidenceScore': newScore,
        'totalPoints': newPoints,
        'completedTasks': completed,
        'lastCompletedDate': todayStr,
        'currentStreak': currentStreak,
        'bestStreak': bestStreak,
      });
    });

    // Sync Local Immediately
    await GlobalSettings.completeTaskLocal(taskId, points);
  }

  Future<bool> buyStreakFreeze(User user, int cost) async {
    try {
      DocumentReference userRef = _db.collection('users').doc(user.uid);
      DocumentSnapshot snap = await userRef.get();
      Map<String, dynamic>? data = snap.data() as Map<String, dynamic>?;
      Map<String, dynamic> safeData = data ?? {};

      int points = safeData['totalPoints'] ?? 0;

      if (points >= cost) {
        await userRef.update({
          'totalPoints': FieldValue.increment(-cost),
          'streakFreezes': FieldValue.increment(1),
        });
        return true;
      }
    } catch (e) {
      print("Buy Freeze Error: $e");
    }
    return false;
  }

  Future<void> equipFreeze(User user) async {
    try {
      DocumentReference userRef = _db.collection('users').doc(user.uid);
      await FirebaseFirestore.instance.runTransaction((transaction) async {
        DocumentSnapshot snap = await transaction.get(userRef);

        // SAFE EXTRACTION: Get the map, allow null, use 0 if missing
        Map<String, dynamic>? data = snap.data() as Map<String, dynamic>?;
        Map<String, dynamic> safeData = data ?? {};

        int owned = safeData['streakFreezes'] ?? 0;
        int active = safeData['activeFreezes'] ?? 0;

        if (owned > 0 && active < 2) {
          transaction.update(userRef, {
            'streakFreezes': FieldValue.increment(-1),
            'activeFreezes': FieldValue.increment(1),
          });
        }
      });
    } catch (e) {
      print("Equip Freeze Error: $e");
    }
  }

  Future<void> saveReflection(
    User user,
    Map<String, dynamic> reflection,
  ) async {
    await _db
        .collection('users')
        .doc(user.uid)
        .collection('history')
        .add(reflection);
  }

  // NEW: Claim Milestone Bonus
  Future<void> claimMilestone(
    User user,
    String milestoneId,
    int bonusPoints,
  ) async {
    DocumentReference userRef = _db.collection('users').doc(user.uid);

    await FirebaseFirestore.instance.runTransaction((transaction) async {
      DocumentSnapshot snap = await transaction.get(userRef);
      if (!snap.exists) throw Exception("User profile missing");

      Map<String, dynamic> data = snap.data() as Map<String, dynamic>;
      List<String> claimed = List<String>.from(
        (data['claimedMilestones'] as List?)?.map((e) => e.toString()) ?? [],
      );
      if (claimed.contains(milestoneId)) return;
      claimed.add(milestoneId);
      double newScore =
          (data['confidenceScore'] as num? ?? 0).toDouble() + bonusPoints;
      int newPoints = (data['totalPoints'] as int? ?? 0) + bonusPoints;
      transaction.update(userRef, {
        'confidenceScore': newScore,
        'totalPoints': newPoints,
        'claimedMilestones': claimed,
      });
    });
  }

  Future<void> resetAllUserData(User user) async {
    final userRef = _db.collection('users').doc(user.uid);

    // 1. Delete the user document and their history subcollection
    final batch = _db.batch();

    // We use a batch to ensure the document and history are deleted together
    // Note: To fully delete a subcollection, you usually have to delete docs one by one,
    // but for a 'Reset', deleting the parent doc is the standard starting point.
    batch.delete(userRef);

    await batch.commit();
  }
}

// --- APP WRAPPER ---
class BloomApp extends StatelessWidget {
  const BloomApp({super.key});

  @override
  Widget build(BuildContext context) {
    return ValueListenableBuilder<Color>(
      valueListenable: GlobalSettings.themeColor,
      builder: (context, color, _) {
        return ValueListenableBuilder<ThemeMode>(
          valueListenable: GlobalSettings.themeMode,
          builder: (context, mode, _) {
            return ValueListenableBuilder<String>(
              valueListenable: GlobalSettings.language,
              builder: (context, lang, _) {
                return MaterialApp(
                  debugShowCheckedModeBanner: false,
                  theme: ThemeData(
                    colorScheme: ColorScheme.fromSeed(
                      seedColor: color,
                      brightness: Brightness.light,
                    ),
                    useMaterial3: true,
                    fontFamily: 'Georgia',
                    pageTransitionsTheme: const PageTransitionsTheme(
                      builders: <TargetPlatform, PageTransitionsBuilder>{
                        TargetPlatform.android:
                            CupertinoPageTransitionsBuilder(),
                        TargetPlatform.iOS: CupertinoPageTransitionsBuilder(),
                        TargetPlatform.macOS: CupertinoPageTransitionsBuilder(),
                        TargetPlatform.windows:
                            FadeUpwardsPageTransitionsBuilder(),
                        TargetPlatform.linux:
                            FadeUpwardsPageTransitionsBuilder(),
                      },
                    ),
                    // SOLID BACKGROUNDS (Fixes "background moving out" glitch)
                    scaffoldBackgroundColor: ColorScheme.fromSeed(
                      seedColor: color,
                      brightness: Brightness.light,
                    ).surface,
                    canvasColor: ColorScheme.fromSeed(
                      seedColor: color,
                      brightness: Brightness.light,
                    ).surface,
                    // Removed duplicate useMaterial3/fontFamily
                  ),

                  darkTheme: ThemeData(
                    colorScheme: ColorScheme.fromSeed(
                      seedColor: color,
                      brightness: Brightness.dark,
                    ),
                    useMaterial3: true,
                    fontFamily: 'Georgia',
                    pageTransitionsTheme: const PageTransitionsTheme(
                      builders: <TargetPlatform, PageTransitionsBuilder>{
                        TargetPlatform.android:
                            CupertinoPageTransitionsBuilder(),
                        TargetPlatform.iOS: CupertinoPageTransitionsBuilder(),
                        TargetPlatform.macOS: CupertinoPageTransitionsBuilder(),
                        TargetPlatform.windows:
                            FadeUpwardsPageTransitionsBuilder(),
                        TargetPlatform.linux:
                            FadeUpwardsPageTransitionsBuilder(),
                      },
                    ),
                    // SOLID BACKGROUNDS (DARK MODE)
                    scaffoldBackgroundColor: ColorScheme.fromSeed(
                      seedColor: color,
                      brightness: Brightness.dark,
                    ).surface,
                    canvasColor: ColorScheme.fromSeed(
                      seedColor: color,
                      brightness: Brightness.dark,
                    ).surface,
                  ),
                  themeMode: mode,
                  builder: (context, child) {
                    return ValueListenableBuilder<String>(
                      valueListenable: GlobalSettings.language,
                      builder: (context, _, __) => child!,
                    );
                  },
                  home: const AuthWrapper(),
                );
              },
            );
          },
        );
      },
    );
  }
}

class AuthWrapper extends StatelessWidget {
  const AuthWrapper({super.key});
  @override
  Widget build(BuildContext context) {
    return StreamBuilder<User?>(
      stream: AuthService().userStream,
      builder: (context, snapshot) {
        if (snapshot.connectionState == ConnectionState.waiting)
          return theLoadingScreen();
        if (snapshot.hasData) {
          return FutureBuilder(
            // Run sequentially: Setup -> Merge -> DailyCheck
            future: (() async {
              await UserService().setupUserProfile(snapshot.data!);
              await _mergeLocalToCloud(snapshot.data!);
              await UserService().handleDailyCheckIn(snapshot.data!);
            })(),
            builder: (context, profileSnap) {
              if (profileSnap.connectionState == ConnectionState.done) {
                return LevelMapScreen(user: snapshot.data!);
              }
              return theLoadingScreen();
            },
          );
        }
        return const AuthScreen();
      },
    );
  }
}

Future<void> _mergeLocalToCloud(User user) async {
  LocalStorageService localStore = LocalStorageService();
  await localStore.init();
  Map<String, dynamic> localData = localStore.loadProfile();

  if (localData.isNotEmpty) {
    await FirebaseFirestore.instance
        .collection('users')
        .doc(user.uid)
        .set(localData, SetOptions(merge: true));
  }
}

Widget theLoadingScreen() => const SplashScreen();

// --- AUTH SCREEN ---
class AuthScreen extends StatefulWidget {
  const AuthScreen({super.key});
  @override
  State<AuthScreen> createState() => _AuthScreenState();
}

class _AuthScreenState extends State<AuthScreen> {
  final TextEditingController _emailController = TextEditingController();
  final TextEditingController _passwordController = TextEditingController();
  bool _isLoginMode = true;
  bool _isLoading = false;

  void _setLoading(bool value) {
    setState(() {
      _isLoading = value;
    });
  }

  void _handleEmailAuth() async {
    _setLoading(true);
    try {
      if (_isLoginMode) {
        await AuthService().signInWithEmail(
          _emailController.text.trim(),
          _passwordController.text.trim(),
        );
      } else {
        await AuthService().signUpWithEmail(
          _emailController.text.trim(),
          _passwordController.text.trim(),
        );
      }
    } catch (e) {
      if (!mounted) return;
      ScaffoldMessenger.of(
        context,
      ).showSnackBar(SnackBar(content: Text(e.toString())));
    } finally {
      if (!mounted) return;
      _setLoading(false);
    }
  }

  void _handleGoogle() async {
    _setLoading(true);
    try {
      await AuthService().signInWithGoogle();
    } catch (e) {
      print(e);
    }
    if (mounted) _setLoading(false);
  }

  void _handleAnonymous() async {
    _setLoading(true);
    try {
      await AuthService().signInAnonymously();
    } catch (e) {
      print(e);
    }
    if (mounted) _setLoading(false);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Stack(
        children: [
          Center(
            child: SingleChildScrollView(
              padding: const EdgeInsets.all(30.0),
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Icon(
                    Icons.wb_sunny,
                    size: 80,
                    color: Theme.of(context).colorScheme.primary,
                  ),
                  const SizedBox(height: 20),
                  Text(
                    _isLoginMode ? "Welcome Back" : "Join Bloom",
                    style: const TextStyle(
                      fontSize: 32,
                      fontWeight: FontWeight.bold,
                    ),
                    textAlign: TextAlign.center,
                  ),
                  const SizedBox(height: 40),
                  TextField(
                    controller: _emailController,
                    decoration: const InputDecoration(
                      labelText: "Email",
                      border: OutlineInputBorder(),
                      prefixIcon: Icon(Icons.email_outlined),
                    ),
                  ),
                  const SizedBox(height: 15),
                  TextField(
                    controller: _passwordController,
                    obscureText: true,
                    decoration: const InputDecoration(
                      labelText: "Password",
                      border: OutlineInputBorder(),
                      prefixIcon: Icon(Icons.lock_outline),
                    ),
                  ),
                  const SizedBox(height: 20),
                  SizedBox(
                    width: double.infinity,
                    child: ElevatedButton(
                      onPressed: _handleEmailAuth,
                      style: ElevatedButton.styleFrom(
                        padding: const EdgeInsets.symmetric(vertical: 15),
                        backgroundColor: Theme.of(context).colorScheme.primary,
                        foregroundColor: Theme.of(context).colorScheme.surface,
                        shape: RoundedRectangleBorder(
                          borderRadius: BorderRadius.circular(15),
                        ),
                      ),
                      child: Text(_isLoginMode ? "Login" : "Create Account"),
                    ),
                  ),
                  TextButton(
                    onPressed: () =>
                        setState(() => _isLoginMode = !_isLoginMode),
                    child: Text(
                      _isLoginMode
                          ? "Don't have an account? Sign Up"
                          : "Already have an account? Sign In",
                    ),
                  ),
                  const SizedBox(height: 30),
                  const Text("OR", style: TextStyle(color: Colors.grey)),
                  const SizedBox(height: 30),
                  SizedBox(
                    width: double.infinity,
                    child: OutlinedButton.icon(
                      onPressed: _handleGoogle,
                      icon: const Icon(Icons.login, color: Colors.red),
                      label: const Text("Sign in with Google"),
                      style: OutlinedButton.styleFrom(
                        padding: const EdgeInsets.symmetric(vertical: 15),
                        shape: RoundedRectangleBorder(
                          borderRadius: BorderRadius.circular(15),
                        ),
                      ),
                    ),
                  ),
                  const SizedBox(height: 15),
                  SizedBox(
                    width: double.infinity,
                    child: TextButton(
                      onPressed: _handleAnonymous,
                      child: const Text("Continue as Guest"),
                    ),
                  ),
                ],
              ),
            ),
          ),
          if (_isLoading)
            Positioned.fill(
              child: Container(
                color: Theme.of(context).colorScheme.surface.withOpacity(
                  0.8,
                ), // Semi-transparent solid
                child: Center(
                  child: Container(
                    padding: const EdgeInsets.all(30),
                    decoration: BoxDecoration(
                      color: Theme.of(context).colorScheme.surface,
                      borderRadius: BorderRadius.circular(20),
                    ),
                    child: Column(
                      mainAxisSize: MainAxisSize.min,
                      children: [
                        const CircularProgressIndicator(color: Colors.teal),
                        const SizedBox(height: 20),
                        Text(
                          "Signing into your account...",
                          style: TextStyle(color: Colors.grey[700]),
                        ),
                      ],
                    ),
                  ),
                ),
              ),
            ),
        ],
      ),
    );
  }
}

// --- LEVEL MAP SCREEN ---
class LevelMapScreen extends StatefulWidget {
  final User user;
  const LevelMapScreen({super.key, required this.user});

  @override
  State<LevelMapScreen> createState() => _LevelMapScreenState();
}

class _LevelMapScreenState extends State<LevelMapScreen> {
  double _confidenceScore = 0;
  int _currentStreak = 0;
  String _userName = "Brave Soul";
  bool _isLoading = true;
  bool _animateIn = false;

  @override
  void initState() {
    super.initState();
    _loadUserData();
  }

  void _loadUserData() async {
    var data = await UserService().getUserData(widget.user);
    if (mounted) {
      setState(() {
        _confidenceScore = (data['confidenceScore'] ?? 0).toDouble();
        _currentStreak = data['currentStreak'] ?? 0;
        _userName = data['userName'] ?? "Brave Soul";
        _isLoading = false;
        _animateIn = true;
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    // 1. Scaffold is ROOT. It never rebuilds unnecessarily.
    return Scaffold(
      // backgroundColor: Theme.of(context).colorScheme.surface,

      // 2. AppBar Title: Listen to BOTH Language + UserProfile
      appBar: AppBar(
        // backgroundColor: Colors.transparent,
        elevation: 0,
        title: ValueListenableBuilder<String>(
          valueListenable: GlobalSettings.language,
          builder: (context, lang, _) {
            return ValueListenableBuilder<Map<String, dynamic>>(
              valueListenable: GlobalSettings.userProfile,
              builder: (_, profile, __) {
                return Text(
                  "${AppTexts.get('hello', lang)}, ${profile['userName'] ?? 'Brave Soul'}!",
                  style: TextStyle(
                    fontWeight: FontWeight.bold,
                    color: Theme.of(context).colorScheme.onSurface,
                  ),
                );
              },
            );
          },
        ),
        actions: [
          IconButton(
            icon: Icon(Icons.analytics_outlined, color: Colors.grey),
            onPressed: () => Navigator.push(
              context,
              MaterialPageRoute(
                builder: (_) => ProgressScreen(user: widget.user),
              ),
            ),
          ),
          IconButton(
            icon: Icon(Icons.person_outline, color: Colors.grey),
            onPressed: () => Navigator.push(
              context,
              MaterialPageRoute(
                builder: (_) => ProfileScreen(user: widget.user),
              ),
            ),
          ),
          IconButton(
            icon: const Icon(Icons.shopping_bag, color: Colors.grey),
            onPressed: () => Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => ShopScreen(user: widget.user)),
            ),
          ),
          IconButton(
            icon: Icon(Icons.logout, color: Colors.grey),
            onPressed: () => AuthService().signOut(),
          ),
        ],
      ),

      // 3. Body: Listen to Language, then Stream Data
      body: ValueListenableBuilder<Map<String, dynamic>>(
        valueListenable: GlobalSettings.userProfile,
        builder: (context, profile, _) {
          return ValueListenableBuilder<String>(
            valueListenable: GlobalSettings.language,
            builder: (context, lang, _) {
              // Use local profile data for instant updates
              double score = (profile['confidenceScore'] ?? 0).toDouble();
              double progress = (score / MAX_CONFIDENCE_SCORE).clamp(
                0.0,
                1.0,
              ); // Uses 880
              int streak = profile['currentStreak'] ?? 0;

              return SingleChildScrollView(
                physics: const BouncingScrollPhysics(),
                padding: const EdgeInsets.symmetric(
                  horizontal: 24.0,
                  vertical: 10,
                ),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    const SizedBox(height: 20),
                    // Growth Gauge
                    GestureDetector(
                      onTap: () => Navigator.push(
                        context,
                        MaterialPageRoute(
                          builder: (_) => MilestoneScreen(user: widget.user),
                        ),
                      ),
                      child: Container(
                        padding: const EdgeInsets.all(24),
                        decoration: BoxDecoration(
                          gradient: LinearGradient(
                            colors: [
                              Theme.of(context).colorScheme.surface,
                              Theme.of(
                                context,
                              ).colorScheme.primary.withOpacity(0.2),
                            ],
                            begin: Alignment.topLeft,
                            end: Alignment.bottomRight,
                          ),
                          borderRadius: BorderRadius.circular(30),
                          boxShadow: [
                            BoxShadow(
                              color: Theme.of(
                                context,
                              ).colorScheme.onSurfaceVariant.withOpacity(0.5),
                              blurRadius: 3,
                            ),
                          ],
                        ),
                        child: Column(
                          children: [
                            Row(
                              mainAxisAlignment: MainAxisAlignment.spaceBetween,
                              children: [
                                Flexible(
                                  // ADD Flexible
                                  child: Tr(
                                    'progress',
                                    style: TextStyle(
                                      color: Theme.of(
                                        context,
                                      ).colorScheme.onSurfaceVariant,
                                      fontSize: 18,
                                      fontWeight: FontWeight.w500,
                                    ),
                                    softWrap: true, // ADD THIS
                                    overflow: TextOverflow.visible, // ADD THIS
                                  ),
                                ),
                                Text(
                                  "${(progress * 100).toInt()}%",
                                  style: TextStyle(
                                    color: Theme.of(
                                      context,
                                    ).colorScheme.onSurfaceVariant,
                                    fontSize: 22,
                                    fontWeight: FontWeight.bold,
                                  ),
                                  softWrap: true, // ADD THIS
                                  overflow: TextOverflow.visible, // ADD THIS
                                ),
                              ],
                            ),
                            const SizedBox(height: 15),
                            ClipRRect(
                              borderRadius: BorderRadius.circular(10),
                              child: LinearProgressIndicator(
                                value: progress,
                                minHeight: 12,
                                backgroundColor: Colors.white.withOpacity(0.3),
                                valueColor: const AlwaysStoppedAnimation<Color>(
                                  Colors.white,
                                ),
                              ),
                            ),
                            const SizedBox(height: 15),
                            Tr(
                              'tap_to_view_journey',
                              style: TextStyle(
                                color: Theme.of(
                                  context,
                                ).colorScheme.onSurfaceVariant,
                                fontSize: 14,
                                fontStyle: FontStyle.italic,
                              ),
                            ),
                          ],
                        ),
                      ),
                    ),
                    const SizedBox(height: 40),
                    Tr(
                      'choose_level',
                      style: TextStyle(
                        fontSize: 22,
                        fontWeight: FontWeight.bold,
                        color: Theme.of(context).colorScheme.onSurfaceVariant,
                      ),
                    ),
                    const SizedBox(height: 20),
                    _levelCard(lang, "Seedling", Icons.circle, Colors.green, 0),
                    _levelCard(
                      lang,
                      "Sprout",
                      Icons.grass,
                      Colors.lightGreen,
                      100,
                    ),
                    _levelCard(
                      lang,
                      "Leaf",
                      Icons.eco,
                      Colors.greenAccent,
                      200,
                    ),
                    _levelCard(lang, "Stem", Icons.nature, Colors.lime, 300),
                    _levelCard(
                      lang,
                      "Bloom",
                      Icons.local_florist,
                      Colors.teal,
                      400,
                    ),
                    const SizedBox(height: 30),
                    SizedBox(
                      width: double.infinity,
                      child: OutlinedButton.icon(
                        onPressed: () => Navigator.push(
                          context,
                          MaterialPageRoute(
                            builder: (_) => HistoryScreen(user: widget.user),
                          ),
                        ),
                        icon: const Icon(Icons.history),
                        label: Tr('view_journey'),
                        style: OutlinedButton.styleFrom(
                          padding: const EdgeInsets.symmetric(vertical: 15),
                          shape: RoundedRectangleBorder(
                            borderRadius: BorderRadius.circular(15),
                          ),
                        ),
                      ),
                    ),
                    const SizedBox(height: 20),
                  ],
                ),
              );
            },
          );
        },
      ),
    );
  }

  // UPDATE _levelCard to accept 'lang' and use Tr for title/subtitle
  // REPLACE _levelCard method in _LevelMapScreenState
  Widget _levelCard(
    String lang,
    String levelKey,
    IconData icon,
    Color color,
    int delay,
  ) {
    String translationKey =
        'level_${levelKey.toLowerCase()}'; // e.g., "level_seedling"

    return TweenAnimationBuilder(
      duration: Duration(milliseconds: 400 + delay),
      tween: Tween<double>(begin: 0, end: 1),
      builder: (context, double value, child) {
        return Transform.scale(
          scale: value,
          child: Opacity(
            opacity: value,
            child: Container(
              margin: const EdgeInsets.only(bottom: 20),
              decoration: BoxDecoration(
                color: Theme.of(context).colorScheme.surface,
                borderRadius: BorderRadius.circular(25),
                border: Border.all(color: color.withOpacity(0.3), width: 1),
                boxShadow: [
                  BoxShadow(
                    color: Theme.of(
                      context,
                    ).colorScheme.onSurface.withOpacity(0.3),
                    blurRadius: 7,
                    offset: const Offset(0, 7),
                  ),
                ],
              ),
              // THE FIX: Material wrapper for InkWell ripple
              child: Material(
                color: Colors.transparent,
                child: ListTile(
                  contentPadding: const EdgeInsets.symmetric(
                    horizontal: 20,
                    vertical: 10,
                  ),
                  leading: buildBloomIcon(icon, color),
                  title: Tr(
                    translationKey,
                    style: const TextStyle(
                      fontSize: 18,
                      fontWeight: FontWeight.bold,
                    ),
                  ),
                  subtitle: Tr(
                    'tap_to_start',
                    style: TextStyle(
                      color: Theme.of(context).colorScheme.onSurfaceVariant,
                      fontSize: 13,
                    ),
                  ),
                  trailing: const Icon(
                    Icons.arrow_forward_ios,
                    size: 16,
                    color: Colors.grey,
                  ),
                  onTap: () async {
                    await Navigator.push(
                      context,
                      MaterialPageRoute(
                        builder: (context) =>
                            TaskScreen(user: widget.user, levelName: levelKey),
                      ),
                    );
                    _loadUserData();
                  },
                ),
              ),
            ),
          ),
        );
      },
    );
  }
}

// --- GAMIFIED PROGRESS SCREEN ---
class ProgressScreen extends StatelessWidget {
  final User user;
  const ProgressScreen({super.key, required this.user});

  Future<void> _showContactDialog(BuildContext context) async {
    const email = 'feedback.bloom@gmail.com';
    final uri = Uri.parse('mailto:$email');
    return showDialog(
      context: context,
      builder: (ctx) => AlertDialog(
        title: Tr('contact_us'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Tr('contact_email_prompt'), // Add this key to AppTexts
            const SizedBox(height: 12),
            InkWell(
              onTap: () async => await launchUrl(uri),
              child: Container(
                padding: const EdgeInsets.symmetric(
                  vertical: 10,
                  horizontal: 12,
                ),
                decoration: BoxDecoration(
                  color: Theme.of(context).colorScheme.primaryContainer,
                  borderRadius: BorderRadius.circular(8),
                ),
                child: Row(
                  mainAxisSize: MainAxisSize.min,
                  children: [
                    Icon(
                      Icons.email,
                      size: 20,
                      color: Theme.of(context).colorScheme.primary,
                    ),
                    const SizedBox(width: 8),
                    Text(
                      email,
                      style: TextStyle(
                        fontSize: 14,
                        fontWeight: FontWeight.w600,
                        color: Theme.of(context).colorScheme.primary,
                        decoration: TextDecoration.underline,
                      ),
                    ),
                  ],
                ),
              ),
            ),
          ],
        ),
        actions: [
          TextButton(onPressed: () => Navigator.pop(ctx), child: Tr('close')),
        ],
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return ValueListenableBuilder<Map<String, dynamic>>(
      valueListenable: GlobalSettings.userProfile,
      builder: (context, profile, _) {
        // SAFE GUARD: Never return loading screen
        final safeProfile = (profile.isEmpty) ? <String, dynamic>{} : profile;

        return ValueListenableBuilder<String>(
          valueListenable: GlobalSettings.language,
          builder: (context, lang, _) {
            double score =
                (safeProfile['confidenceScore'] as num?)?.toDouble() ?? 0;
            int streak = profile['currentStreak'] as int? ?? 0;
            String name = profile['userName'] ?? "Brave Soul";
            String rank = _getRank(score);

            return Scaffold(
              appBar: AppBar(title: Tr('your_growth_path'), elevation: 0),
              body: SingleChildScrollView(
                padding: const EdgeInsets.all(24.0),
                child: Column(
                  children: [
                    const SizedBox(height: 20),
                    Center(
                      child: Column(
                        children: [
                          CircleAvatar(
                            radius: 60,
                            backgroundColor: Theme.of(
                              context,
                            ).colorScheme.primary,
                            child: Icon(
                              Icons.person,
                              size: 60,
                              color: Theme.of(context).colorScheme.surface,
                            ),
                          ),
                          const SizedBox(height: 15),
                          Text(
                            name,
                            style: const TextStyle(
                              fontSize: 24,
                              fontWeight: FontWeight.bold,
                            ),
                          ),
                          Text(
                            "Confident Grower",
                            style: TextStyle(color: Colors.grey[600]),
                          ),
                        ],
                      ),
                    ),
                    const SizedBox(height: 40),
                    Wrap(
                      spacing: 20,
                      runSpacing: 20,
                      alignment: WrapAlignment.center,
                      children: [
                        // USE AppTexts.get('key', lang) -> Returns String
                        _statCard(
                          context,
                          AppTexts.get('total_points', lang),
                          score.toInt().toString(),
                          Icons.star,
                          Colors.amber,
                        ),
                        _statCard(
                          context,
                          AppTexts.get('current_streak', lang),
                          "$streak ${AppTexts.get('days', lang)}",
                          Icons.fireplace,
                          Colors.orange,
                        ),
                        _statCard(
                          context,
                          AppTexts.get('tasks_done', lang),
                          ((profile['completedTasks'] as List?)?.length ?? 0)
                              .toString(),
                          Icons.check_circle,
                          Colors.green,
                        ),
                        _statCard(
                          context,
                          AppTexts.get('rank', lang),
                          rank,
                          Icons.emoji_events,
                          Colors.blue,
                        ),
                      ],
                    ),
                    const SizedBox(height: 40),
                    Tr(
                      'your_journey',
                      style: const TextStyle(
                        fontSize: 20,
                        fontWeight: FontWeight.bold,
                      ),
                    ),
                    const SizedBox(height: 10),
                    Tr(
                      'keep_growing_sub',
                      textAlign: TextAlign.center,
                      style: TextStyle(color: Colors.grey),
                    ),
                    const SizedBox(height: 40),
                    OutlinedButton.icon(
                      onPressed: () => Navigator.push(
                        context,
                        MaterialPageRoute(builder: (_) => FAQScreen()),
                      ),
                      icon: const Icon(Icons.help_outline),
                      label: Tr('how_it_works'),
                    ),
                    const SizedBox(height: 15), // Spacing
                    // --- ADD THIS CONTACT BUTTON ---
                    TextButton.icon(
                      onPressed: () => _showContactDialog(context),
                      icon: const Icon(Icons.contact_mail_outlined),
                      label: Tr('contact_us'),
                      style: TextButton.styleFrom(
                        // Removes standard button internal padding so it scales naturally like the FAQ button
                        padding: EdgeInsets.symmetric(
                          horizontal: 12.0,
                          vertical: 8.0,
                        ),
                      ),
                    ),
                  ],
                ),
              ),
            );
          },
        );
      },
    );
  }

  String _getRank(double score) {
    if (score < 40) return "Seedling";
    if (score < 140) return "Sprout";
    if (score < 340) return "Leaf";
    if (score < 580) return "Stem";
    return "Bloom";
  }

  // Helper _statCard needs to accept String for label/value now
  Widget _statCard(
    BuildContext context,
    String label,
    String value,
    IconData icon,
    Color color,
  ) {
    return SizedBox(
      width: (MediaQuery.of(context).size.width / 2) - 32,
      child: Container(
        padding: const EdgeInsets.all(20),
        decoration: BoxDecoration(
          color: Theme.of(context).colorScheme.surface,
          borderRadius: BorderRadius.circular(20),
          border: Border.all(
            color: Theme.of(context).colorScheme.primary.withOpacity(0.1),
            width: 1,
          ),
          boxShadow: [
            BoxShadow(
              color: Theme.of(context).colorScheme.onSurface.withOpacity(0.5),
              blurRadius: 7,
              offset: const Offset(0, 7),
            ),
          ],
        ),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(icon, color: color, size: 30),
            const SizedBox(height: 10),
            Text(
              value,
              style: const TextStyle(fontSize: 18, fontWeight: FontWeight.bold),
              textAlign: TextAlign.center,
            ),
            Text(
              label,
              style: const TextStyle(fontSize: 12, color: Colors.grey),
              textAlign: TextAlign.center,
            ),
          ],
        ),
      ),
    );
  }
}

// --- TASK SCREEN ---
class TaskScreen extends StatefulWidget {
  final User user;
  final String levelName;
  const TaskScreen({super.key, required this.user, required this.levelName});

  @override
  State<TaskScreen> createState() => _TaskScreenState();
}

class _TaskScreenState extends State<TaskScreen> {
  Map<String, String>? _currentTask;
  bool _isCompleted = false;
  bool _isLoading = true;
  int _levelTaskCount = 0;
  bool _isPickingTask = false;

  @override
  void initState() {
    super.initState();
    _loadLevelProgress();
    _pickRandomTask();
  }

  void _loadLevelProgress() {
    final Map<String, dynamic> levelProgress =
        GlobalSettings.userProfile.value['levelTaskCounts'] ?? {};
    _levelTaskCount = levelProgress[widget.levelName] ?? 0;
  }

  // 1. DECLARE HELPER METHODS FIRST (before they're used)
  double _getLevelPoints(String level) {
    switch (level) {
      case 'Seedling':
        return 2.0;
      case 'Sprout':
        return 5.0;
      case 'Leaf':
        return 10.0;
      case 'Stem':
        return 12.0;
      case 'Bloom':
        return 15.0;
      default:
        return 5.0;
    }
  }

  void _saveLevelProgress() {
    var profile = Map<String, dynamic>.from(GlobalSettings.userProfile.value);
    Map<String, dynamic> levelCounts = Map<String, dynamic>.from(
      GlobalSettings.userProfile.value['levelTaskCounts'] ?? {},
    );
    levelCounts[widget.levelName] = _levelTaskCount;
    GlobalSettings.userProfile.value = {
      ...GlobalSettings.userProfile.value,
      'levelTaskCounts': levelCounts,
    };
    GlobalSettings._storage.saveProfile(GlobalSettings.userProfile.value);
  }

  void _showLevelUpSuggestion() {
    if (!mounted) return;

    showDialog(
      context: context,
      barrierDismissible: false,
      builder: (ctx) => AlertDialog(
        title: Tr('level_up_suggestion_title'),
        content: Tr('level_up_suggestion_message'),
        actions: [
          TextButton(
            onPressed: () {
              Navigator.pop(context);
              _pickRandomTask();
            },
            child: Tr('stay_here'),
          ),
          ElevatedButton(
            onPressed: () {
              Navigator.pop(context);
              Navigator.pop(context);
            },
            child: Tr('move_to_next_level'),
          ),
        ],
      ),
    );
  }

  void _pickRandomTask() async {
    if (!mounted || _isPickingTask) return;
    _isPickingTask = true;

    // Only show loading on INITIAL load
    if (_currentTask == null && mounted) {
      setState(() => _isLoading = true);
    }

    try {
      String lang = GlobalSettings.language.value;
      debugPrint("🎲 Picking random task for ${widget.levelName}...");

      var userData = await UserService()
          .getUserData(widget.user)
          .timeout(
            const Duration(seconds: 5),
            onTimeout: () {
              debugPrint("⚠️ Firestore timeout, using empty completed tasks");
              return <String, dynamic>{};
            },
          );

      List completedTasks = List<String>.from(
        (userData['completedTasks'] as List?)?.map((e) => e.toString()) ?? [],
      );

      List<Map<String, String>> allTasks = TaskLibrary.getTasks(
        widget.levelName,
        GlobalSettings.language.value,
      );

      if (allTasks.isEmpty) {
        debugPrint("❌ No tasks found for level: ${widget.levelName}");
        if (mounted) setState(() => _isLoading = false);
        return;
      }

      List<Map<String, String>> availableTasks = allTasks
          .where((t) => !completedTasks.contains(t['id']))
          .toList();

      if (mounted) {
        setState(() {
          if (availableTasks.isNotEmpty) {
            _currentTask = (availableTasks..shuffle()).first;
            debugPrint("✅ New task picked: ${_currentTask!['id']}");
          } else {
            _currentTask = (allTasks..shuffle()).first; // RECYCLE
            debugPrint("🔄 Recycling tasks - all completed");
          }
          _isLoading = false;
        });
      }
    } on Exception catch (err) {
      debugPrint("❌ Error picking task: $err");
      if (mounted) setState(() => _isLoading = false);
    } finally {
      _isPickingTask = false;
    }
  }

  void _completeTask() {
    if (_currentTask == null) return;
    setState(() => _isLoading = true);

    Navigator.push(
      context,
      MaterialPageRoute(
        builder: (context) => ReflectionScreen(
          user: widget.user,
          taskTitle: _currentTask!['title']!,
          taskId: _currentTask!['id']!,
          levelName: widget.levelName,
          onDone: _onReflectionDone,
        ),
      ),
    ).then((result) {
      // result == true  -> Finish pressed: _onReflectionDone() already picked NEW task
      // result != true  -> Back pressed: Do NOTHING (keep SAME task)
      if (result != true) {
        debugPrint("↩️ Back pressed in Reflection, KEEPING current task");
        // Don't call _pickRandomTask() - keep the current task
      } else {
        debugPrint(
          "✅ Finish pressed, _onReflectionDone already handled new task",
        );
      }
    });
  }

  void _onReflectionDone() {
    _levelTaskCount++;
    _saveLevelProgress();
    debugPrint(
      "✅ Task completed. Level count for ${widget.levelName}: $_levelTaskCount",
    );

    if (_levelTaskCount > 0 && _levelTaskCount % 10 == 0) {
      Future.microtask(() => _showLevelUpSuggestion());
    }

    // Pick NEW task after completion
    _pickRandomTask();
  }

  @override
  Widget build(BuildContext context) {
    // Only show full loading screen on INITIAL load, not on return
    if (_isLoading && _currentTask == null) return theLoadingScreen();

    return ValueListenableBuilder<String>(
      valueListenable: GlobalSettings.language,
      builder: (context, lang, _) {
        if (_currentTask == null) return theLoadingScreen();

        final m = _currentTask!;
        final userName =
            GlobalSettings.userProfile.value['userName'] ?? 'Brave Soul';

        return Scaffold(
          appBar: AppBar(
            title: Text(
              AppTexts.get('stage_label', lang).replaceAll(
                '{0}',
                AppTexts.get('level_${widget.levelName.toLowerCase()}', lang),
              ),
            ),
            elevation: 0,
          ),
          body: SingleChildScrollView(
            padding: const EdgeInsets.all(24.0),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(
                  AppTexts.get(
                    'keep_growing',
                    lang,
                  ).replaceAll('{0}', userName),
                  style: const TextStyle(
                    fontSize: 22,
                    fontWeight: FontWeight.bold,
                  ),
                ),
                const SizedBox(height: 10),
                Text(
                  AppTexts.get('current_challenge', lang),
                  style: TextStyle(color: Colors.grey[600]),
                ),
                const SizedBox(height: 24),

                // CARD LAYOUT
                Card(
                  elevation: 4,
                  shape: RoundedRectangleBorder(
                    borderRadius: BorderRadius.circular(20),
                  ),
                  child: AnimatedContainer(
                    duration: const Duration(milliseconds: 500),
                    padding: const EdgeInsets.all(24),
                    decoration: BoxDecoration(
                      borderRadius: BorderRadius.circular(20),
                      color: _isCompleted
                          ? Colors.green.withOpacity(0.1)
                          : Theme.of(
                              context,
                            ).colorScheme.surfaceContainerHighest,
                      border: Border.all(
                        color: _isCompleted ? Colors.green : Colors.transparent,
                        width: 2,
                      ),
                    ),
                    child: Column(
                      children: [
                        Icon(
                          _isCompleted
                              ? Icons.check_circle
                              : Icons.wb_sunny_outlined,
                          color: _isCompleted ? Colors.green : Colors.orange,
                          size: 50,
                        ),
                        const SizedBox(height: 16),
                        Text(
                          _currentTask!['title']!,
                          style: const TextStyle(
                            fontSize: 20,
                            fontWeight: FontWeight.bold,
                          ),
                          textAlign: TextAlign.center,
                        ),
                        const SizedBox(height: 12),
                        Text(
                          TaskLibrary.getTaskById(
                                _currentTask!['id']!,
                                GlobalSettings.language.value,
                              )?['desc'] ??
                              _currentTask!['desc']!,
                          style: const TextStyle(fontSize: 16, height: 1.5),
                          textAlign: TextAlign.center,
                        ),
                        const SizedBox(height: 24),
                        SizedBox(
                          width: double.infinity,
                          child: ElevatedButton(
                            onPressed: _isCompleted ? null : _completeTask,
                            style: ElevatedButton.styleFrom(
                              padding: const EdgeInsets.symmetric(vertical: 16),
                              backgroundColor: Theme.of(
                                context,
                              ).colorScheme.primary,
                              foregroundColor: Theme.of(
                                context,
                              ).colorScheme.onPrimary,
                              shape: RoundedRectangleBorder(
                                borderRadius: BorderRadius.circular(16),
                              ),
                            ),
                            child: Text(
                              _isCompleted
                                  ? AppTexts.get("well_done", lang)
                                  : AppTexts.get("i_completed", lang),
                              style: const TextStyle(
                                fontSize: 18,
                                fontWeight: FontWeight.bold,
                              ),
                            ),
                          ),
                        ),
                      ],
                    ),
                  ),
                ),
              ],
            ),
          ),
        );
      },
    );
  }
}

// --- REFLECTION SCREEN ---
class ReflectionScreen extends StatefulWidget {
  final User user;
  final String taskTitle;
  final String taskId;
  final String levelName;
  final VoidCallback onDone;
  const ReflectionScreen({
    super.key,
    required this.user,
    required this.taskTitle,
    required this.taskId,
    required this.levelName,
    required this.onDone,
  });

  @override
  State<ReflectionScreen> createState() => _ReflectionScreenState();
}

class _ReflectionScreenState extends State<ReflectionScreen> {
  double _anxiety = 5;
  final TextEditingController _noteController = TextEditingController();
  bool _isSaving = false;

  // Update _finishReflection to return result
  void _finishReflection() async {
    setState(() => _isSaving = true);
    double points = 0;
    if (widget.levelName == "Seedling")
      points = 2.0;
    else if (widget.levelName == "Sprout")
      points = 5.0;
    else if (widget.levelName == "Leaf")
      points = 10.0;
    else if (widget.levelName == "Stem")
      points = 12.0;
    else if (widget.levelName == "Bloom")
      points = 15.0;

    try {
      await UserService()
          .completeTask(widget.user, widget.taskId, points)
          .timeout(const Duration(seconds: 10));

      await UserService()
          .saveReflection(widget.user, {
            'taskId': widget.taskId,
            'title': widget.taskTitle,
            'anxiety': _anxiety.toInt(),
            'note': _noteController.text,
            'date': DateFormat('MMM d, yyyy - hh:mm a').format(DateTime.now()),
          })
          .timeout(const Duration(seconds: 10));
    } on TimeoutException catch (e) {
      debugPrint("⏱️ Reflection timeout: $e");
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Network timeout. Progress saved locally.')),
        );
      }
    } catch (e) {
      debugPrint("❌ Reflection error: $e");
      if (mounted) {
        ScaffoldMessenger.of(
          context,
        ).showSnackBar(SnackBar(content: Text('Error saving: $e')));
      }
    } finally {
      if (mounted) setState(() => _isSaving = false);
    }

    // Return TRUE to indicate completion (Finish pressed)
    if (!mounted) return;
    widget.onDone(); // Calls _onReflectionDone() which picks new task
    Navigator.pop(context, true); // true = Finish pressed
  }

  // Inside _ReflectionScreenState.build
  @override
  Widget build(BuildContext context) {
    // ROOT SCAFFOLD - No PopScope needed
    return Scaffold(
      appBar: AppBar(
        title: Tr('reflect'),
        elevation: 0,
        leading: IconButton(
          icon: const Icon(Icons.arrow_back),
          onPressed: () =>
              Navigator.pop(context, false), // ← false = Back pressed
        ),
      ),
      // BODY LISTENS TO LANGUAGE
      body: ValueListenableBuilder<String>(
        valueListenable: GlobalSettings.language,
        builder: (context, lang, _) {
          return SingleChildScrollView(
            padding: const EdgeInsets.all(30.0),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Tr(
                  'reflect',
                  style: const TextStyle(
                    fontSize: 28,
                    fontWeight: FontWeight.bold,
                  ),
                ),
                Text(
                  "${AppTexts.get("challenge", lang)}: ${widget.taskTitle}",
                  style: TextStyle(fontSize: 16, color: Colors.grey[600]),
                ),
                const SizedBox(height: 40),

                // Anxiety Section
                Container(
                  padding: const EdgeInsets.all(20),
                  decoration: BoxDecoration(
                    color: Theme.of(
                      context,
                    ).colorScheme.surfaceContainerHighest,
                    borderRadius: BorderRadius.circular(20),
                  ),
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      Tr(
                        'anxiety_q',
                        style: const TextStyle(
                          fontSize: 16,
                          fontWeight: FontWeight.w500,
                        ),
                      ),
                      Slider(
                        value: _anxiety,
                        min: 1,
                        max: 10,
                        divisions: 9,
                        activeColor: Theme.of(context).colorScheme.primary,
                        label: _anxiety.round().toString(),
                        onChanged: (v) => setState(() => _anxiety = v),
                      ),
                      Center(
                        child: Text(
                          "${_anxiety.toInt()}",
                          style: const TextStyle(
                            fontSize: 24,
                            fontWeight: FontWeight.bold,
                            color: Colors.teal,
                          ),
                        ),
                      ),
                    ],
                  ),
                ),
                const SizedBox(height: 30),

                // Reflection Text Field
                Tr(
                  'what_happened',
                  style: const TextStyle(
                    fontSize: 16,
                    fontWeight: FontWeight.w500,
                  ),
                ),
                const SizedBox(height: 15),
                TextField(
                  controller: _noteController,
                  maxLines: 5,
                  decoration: InputDecoration(
                    hintText: AppTexts.get('write_experience_hint', lang),
                    border: OutlineInputBorder(
                      borderRadius: BorderRadius.circular(20),
                    ),
                    fillColor: Theme.of(
                      context,
                    ).colorScheme.surfaceContainerHighest,
                    filled: true,
                  ),
                ),
                const SizedBox(height: 40),

                // Finish Button - Returns true = completed
                SizedBox(
                  width: double.infinity,
                  child: _isSaving
                      ? const Center(child: CircularProgressIndicator())
                      : ElevatedButton(
                          onPressed: _finishReflection,
                          style: ElevatedButton.styleFrom(
                            padding: const EdgeInsets.symmetric(vertical: 18),
                            backgroundColor: Theme.of(
                              context,
                            ).colorScheme.primary,
                            foregroundColor: Theme.of(
                              context,
                            ).colorScheme.surface,
                            shape: RoundedRectangleBorder(
                              borderRadius: BorderRadius.circular(20),
                            ),
                          ),
                          child: Tr(
                            'finish',
                            style: const TextStyle(fontSize: 18),
                          ),
                        ),
                ),
              ],
            ),
          );
        },
      ),
    );
  }
}

// --- PROFILE SCREEN ---
class ProfileScreen extends StatefulWidget {
  final User user;
  const ProfileScreen({super.key, required this.user});

  @override
  State<ProfileScreen> createState() => _ProfileScreenState();
}

class _ProfileScreenState extends State<ProfileScreen> {
  String _userName = "";
  final TextEditingController _nameController = TextEditingController();

  @override
  void initState() {
    super.initState();
    _loadData();
  }

  void _loadData() async {
    // Use the service directly for initial load
    var data = await UserService().getUserData(widget.user);
    if (mounted) {
      setState(() {
        _userName = data['userName'] ?? "Brave Soul";
        _nameController.text = _userName;
      });
    }
  }

  void _updateName() async {
    await GlobalSettings.updateName(_nameController.text);
    setState(() => _userName = _nameController.text);
    if (mounted) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Tr('profile_updated')), // WRAP IN SNACKBAR
      );
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Tr('profile'), // Widget
        elevation: 0,
        // backgroundColor: Colors.transparent,
      ),
      // LISTEN TO LANGUAGE FOR THE WHOLE BODY
      body: ValueListenableBuilder<String>(
        valueListenable: GlobalSettings.language,
        builder: (context, lang, _) {
          return SingleChildScrollView(
            padding: const EdgeInsets.all(24.0),
            child: Column(
              children: [
                const SizedBox(height: 20),
                CircleAvatar(
                  radius: 60,
                  backgroundColor: Theme.of(context).colorScheme.primary,
                  child: Icon(
                    Icons.person,
                    size: 60,
                    color: Theme.of(context).colorScheme.surface,
                  ),
                ),
                const SizedBox(height: 20),
                // UserName updates via GlobalSettings.userProfile listener
                ValueListenableBuilder<Map<String, dynamic>>(
                  valueListenable: GlobalSettings.userProfile,
                  builder: (_, p, __) => Text(
                    p['userName'] ?? "Brave Soul",
                    style: const TextStyle(
                      fontSize: 26,
                      fontWeight: FontWeight.bold,
                    ),
                  ),
                ),
                Text(
                  "Confident Grower",
                  style: TextStyle(color: Colors.grey[600]),
                ),
                const SizedBox(height: 40),

                // USE AppTexts.get('key', lang) FOR HELPER METHODS REQUIRING STRINGS
                _buildSection(AppTexts.get('account', lang), [
                  TextField(
                    controller: _nameController,
                    decoration: InputDecoration(
                      labelText: AppTexts.get('display_name', lang),
                      border: const OutlineInputBorder(),
                    ),
                  ),
                  const SizedBox(height: 15),
                  SizedBox(
                    width: double.infinity,
                    child: ElevatedButton(
                      onPressed: _updateName,
                      style: ElevatedButton.styleFrom(
                        backgroundColor: Theme.of(context).colorScheme.primary,
                        foregroundColor: Theme.of(context).colorScheme.surface,
                      ),
                      child: Tr('save_name'), // Widget
                    ),
                  ),
                ]),
                const SizedBox(height: 20),

                _buildSection(AppTexts.get('app_theme', lang), [
                  Tr(
                    'select_color',
                    style: TextStyle(fontSize: 14, fontWeight: FontWeight.w500),
                  ), // Widget
                  const SizedBox(height: 15),
                  Wrap(
                    spacing: 12,
                    runSpacing: 12,
                    children: [
                      ...GlobalSettings.themePalette.map((color) {
                        bool isSelected =
                            GlobalSettings.themeColor.value == color;
                        return GestureDetector(
                          onTap: () => GlobalSettings.saveTheme(color),
                          child: Container(
                            width: 40,
                            height: 40,
                            decoration: BoxDecoration(
                              color: color,
                              shape: BoxShape.circle,
                              border: Border.all(
                                color: isSelected
                                    ? Colors.black
                                    : Colors.transparent,
                                width: 3,
                              ),
                              boxShadow: [
                                BoxShadow(
                                  color: Theme.of(
                                    context,
                                  ).colorScheme.onSurface,
                                  blurRadius: 4,
                                ),
                              ],
                            ),
                            child: isSelected
                                ? const Icon(
                                    Icons.check,
                                    color: Colors.white,
                                    size: 20,
                                  )
                                : null,
                          ),
                        );
                      }).toList(),
                      GestureDetector(
                        onTap: _showColorPicker,
                        child: Container(
                          width: 40,
                          height: 40,
                          decoration: BoxDecoration(
                            shape: BoxShape.circle,
                            gradient: const SweepGradient(
                              colors: [
                                Colors.red,
                                Colors.yellow,
                                Colors.green,
                                Colors.blue,
                                Colors.purple,
                                Colors.red,
                              ],
                            ),
                            border: Border.all(
                              color: Colors.grey.shade300,
                              width: 2,
                            ),
                          ),
                          child: const Icon(
                            Icons.colorize,
                            size: 20,
                            color: Colors.white,
                          ),
                        ),
                      ),
                    ],
                  ),
                  const SizedBox(height: 20),
                  Row(
                    mainAxisAlignment: MainAxisAlignment.spaceAround,
                    children: [
                      _themeModeButton(
                        AppTexts.get('light_mode', lang),
                        ThemeMode.light,
                        lang,
                      ), // Pass String
                      _themeModeButton(
                        AppTexts.get('dark_mode', lang),
                        ThemeMode.dark,
                        lang,
                      ),
                    ],
                  ),
                ]),
                const SizedBox(height: 20),

                _buildSection(AppTexts.get('language', lang), [
                  _buildLanguageSelector(context, lang), // Pass lang
                ]),
                const SizedBox(height: 60),
                TextButton(
                  onPressed: () => AuthService().signOut(),
                  child: Tr(
                    'logout',
                    style: TextStyle(
                      color: Colors.red,
                      fontWeight: FontWeight.bold,
                    ),
                  ),
                ),
                const SizedBox(height: 10), // Spacing
                // --- ADD THIS RESET BUTTON ---
                TextButton(
                  onPressed: () => _showResetConfirmation(context),
                  child: Tr(
                    'reset_all_progress',
                    style: TextStyle(
                      color: Colors.red,
                      fontWeight: FontWeight.bold,
                    ),
                  ),
                ),
                const SizedBox(height: 20), // Bottom padding
              ],
            ),
          );
        },
      ),
    );
  }

  // HELPERS NOW ACCEPT 'lang' STRING

  void _showColorPicker() {
    showDialog(
      context: context,
      builder: (BuildContext context) {
        return AlertDialog(
          title: Tr('pick_theme_color'), // Use Tr widget
          content: SingleChildScrollView(
            child: ColorPicker(
              // HUE WHEEL PICKER
              pickerColor: GlobalSettings.themeColor.value,
              onColorChanged: (color) {
                GlobalSettings.saveTheme(color);
                setState(() {}); // Update dialog preview
              },
              // Optional: Enable alpha slider if you want transparency support
              enableAlpha: false,
              // Optional: Show hex input field
              displayThumbColor: true,
              portraitOnly: true,
            ),
          ),
          actions: [
            TextButton(
              onPressed: () => Navigator.pop(context),
              child: Tr('done'),
            ),
          ],
        );
      },
    );
  }

  Widget _themeModeButton(String label, ThemeMode mode, String lang) {
    bool isSelected = GlobalSettings.themeMode.value == mode;
    return ElevatedButton(
      onPressed: () {
        GlobalSettings.saveThemeMode(mode);
        setState(() {});
      },
      style: ElevatedButton.styleFrom(
        backgroundColor: isSelected
            ? Theme.of(context).colorScheme.primary
            : Colors.grey[200],
        foregroundColor: isSelected
            ? Theme.of(context).colorScheme.surface
            : Colors.black,
      ),
      child: Text(label), // Use passed String
    );
  }

  Widget _buildLanguageSelector(BuildContext context, String lang) {
    // 1. EXPLICITLY TYPE THE LIST
    final List<Map<String, String>> languages = [
      {'code': 'ar', 'name': 'Arabic (عربي)', 'flag': '🇸🇦'},
      {'code': 'bn', 'name': 'Bangla (বাংলা)', 'flag': '🇧🇩'},
      {'code': 'zh', 'name': 'Chinese (简体中文)', 'flag': '🇨🇳'},
      {'code': 'nl', 'name': 'Dutch (Nederlands)', 'flag': '🇳🇱'},
      {'code': 'en', 'name': 'English', 'flag': '🇺🇸'},
      {'code': 'fr', 'name': 'French (Français)', 'flag': '🇫🇷'},
      {'code': 'de', 'name': 'German (Deutsch)', 'flag': '🇩🇪'},
      {'code': 'gu', 'name': 'Gujarati (ગુજરાતી)', 'flag': '🇮🇳'},
      {'code': 'hi', 'name': 'Hindi (हिन्दी)', 'flag': '🇮🇳'},
      {'code': 'id', 'name': 'Indonesian (Bahasa Indonesia)', 'flag': '🇮🇩'},
      {'code': 'ga', 'name': 'Irish (Gaeilge)', 'flag': '🇮🇪'},
      {'code': 'it', 'name': 'Italian (Italiana)', 'flag': '🇮🇹'},
      {'code': 'ja', 'name': 'Japanese (日本語)', 'flag': '🇯🇵'},
      {'code': 'kn', 'name': 'Kannada (ಕನ್ನಡ)', 'flag': '🇮🇳'},
      {'code': 'ko', 'name': 'Korean (한국인)', 'flag': '🇰🇷'},
      {'code': 'my', 'name': 'Malay (Melayu)', 'flag': '🇲🇾'},
      {'code': 'ml', 'name': 'Malayalam (മലയാളം)', 'flag': '🇮🇳'},
      {'code': 'mni', 'name': 'Manipuri ((ꯃեꯏꯇꯩꯂꯣꯟ)', 'flag': '🇮🇳'},
      {'code': 'mr', 'name': 'Marathi (मराठी)', 'flag': '🇮🇳'},
      {'code': 'no', 'name': 'Norwegian (Norsk)', 'flag': '🇳🇴'},
      {'code': 'pt', 'name': 'Portuguese (Português)', 'flag': '🇵🇹'},
      {'code': 'pa', 'name': 'Punjabi (ਪੰਜਾਬੀ)', 'flag': '🇮🇳'},
      {'code': 'ru', 'name': 'Russian (Русскийzzz)', 'flag': '🇷🇺'},
      {'code': 'es', 'name': 'Spanish (Español)', 'flag': '🇪🇸'},
      {'code': 'sv', 'name': 'Swedish (Svenska)', 'flag': '🇸🇪'},
      {'code': 'ta', 'name': 'Tamil (தமிழ்)', 'flag': '🇱🇰'},
      {'code': 'te', 'name': 'Telugu (తెలుగు)', 'flag': '🇮🇳'},
      {'code': 'th', 'name': 'Thai (ภาษาไทย)', 'flag': '🇹🇭'},
      {'code': 'tr', 'name': 'Turkish (Türkçe)', 'flag': '🇹🇷'},
      {'code': 'ur', 'name': 'Urdu (اردو)', 'flag': '🇵🇰'},
      {'code': 'vi', 'name': 'Vietnamese (Tiếng Việt)', 'flag': '🇻🇳'},
    ];

    return Container(
      decoration: BoxDecoration(
        color: Theme.of(context).colorScheme.surface,
        borderRadius: BorderRadius.circular(12),
        border: Border.all(
          color: Theme.of(context).colorScheme.outline.withOpacity(0.2),
        ),
      ),
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
      child: DropdownButtonHideUnderline(
        child: DropdownButton<String>(
          value: lang,
          isExpanded: true,
          icon: Icon(
            Icons.language,
            color: Theme.of(context).colorScheme.primary,
          ),
          dropdownColor: Theme.of(context).colorScheme.surface,
          style: TextStyle(
            fontSize: 16,
            color: Theme.of(context).colorScheme.onSurface,
          ),
          // 2. EXPLICITLY TYPE THE MAP FUNCTION RETURN TYPE
          items: languages.map<DropdownMenuItem<String>>((
            Map<String, String> l,
          ) {
            return DropdownMenuItem<String>(
              value: l['code'],
              child: Row(
                children: [
                  Text(l['flag']!, style: const TextStyle(fontSize: 20)),
                  const SizedBox(width: 12),
                  Text(l['name']!),
                ],
              ),
            );
          }).toList(),
          onChanged: (String? val) {
            if (val != null) {
              GlobalSettings.saveLanguage(val);
            }
          },
        ),
      ),
    );
  }

  Widget _buildSection(String title, List<Widget> children) {
    // Title is now String
    return Container(
      padding: const EdgeInsets.all(20),
      decoration: BoxDecoration(
        color: Theme.of(context).colorScheme.surfaceContainerHighest,
        borderRadius: BorderRadius.circular(25),
        border: Border.all(color: Colors.black.withOpacity(0.05)),
      ),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text(
            title,
            style: const TextStyle(fontSize: 18, fontWeight: FontWeight.bold),
          ), // Use String directly
          const SizedBox(height: 15),
          ...children,
        ],
      ),
    );
  }

  void _showResetConfirmation(BuildContext context) {
    showDialog(
      context: context,
      builder: (ctx) => AlertDialog(
        title: Tr('reset_confirm_title'), // Add key
        content: Tr('reset_confirm_message'), // Add key
        actions: [
          TextButton(onPressed: () => Navigator.pop(ctx), child: Tr('cancel')),
          TextButton(
            onPressed: () async {
              Navigator.pop(ctx);
              await UserService().resetAllUserData(widget.user);
              await GlobalSettings.resetLocalSettings();
              await AuthService().signOut();
              if (mounted)
                ScaffoldMessenger.of(
                  context,
                ).showSnackBar(SnackBar(content: Tr('reset_success')));
            },
            child: Tr('reset_everything', style: TextStyle(color: Colors.red)),
          ),
        ],
      ),
    );
  }
}

// --- HISTORY SCREEN ---
// --- HISTORY SCREEN ---
class HistoryScreen extends StatelessWidget {
  final User user;
  const HistoryScreen({super.key, required this.user});

  // 1. Logic to load history from Firestore
  Future<List<Map<String, dynamic>>> _loadHistory() async {
    QuerySnapshot snap = await FirebaseFirestore.instance
        .collection('users')
        .doc(user.uid)
        .collection('history')
        .orderBy('date', descending: true)
        .get();
    return snap.docs.map((doc) => doc.data() as Map<String, dynamic>).toList();
  }

  // 2. The Premium Popup Logic
  void _showTaskDetail(BuildContext context, Map<String, dynamic> entry) {
    String lang = GlobalSettings.language.value;
    var taskInfo = TaskLibrary.getTaskById(entry['taskId'] ?? "", lang);
    String description = taskInfo?['desc'] ?? "Description not available";

    showGeneralDialog(
      context: context,
      barrierDismissible: true,
      barrierLabel: "Dismiss",
      transitionDuration: const Duration(milliseconds: 400),
      pageBuilder: (context, anim1, anim2) {
        return Align(
          alignment: Alignment.center,
          child: Material(
            color: Colors.transparent,
            child: Container(
              margin: const EdgeInsets.symmetric(horizontal: 30),
              padding: const EdgeInsets.all(24),
              decoration: BoxDecoration(
                color: Theme.of(context).colorScheme.surface,
                borderRadius: BorderRadius.circular(30),
                boxShadow: [BoxShadow(color: Colors.black26, blurRadius: 20)],
              ),
              child: Column(
                mainAxisSize: MainAxisSize.min,
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Row(
                    children: [
                      Icon(
                        Icons.auto_awesome,
                        color: Theme.of(context).colorScheme.onSurfaceVariant,
                        size: 24,
                      ),
                      const SizedBox(width: 10),
                      Expanded(
                        child: Text(
                          entry['title'] ?? "Growth Memory",
                          style: const TextStyle(
                            fontSize: 20,
                            fontWeight: FontWeight.bold,
                          ),
                        ),
                      ),
                      IconButton(
                        icon: const Icon(Icons.close),
                        onPressed: () => Navigator.pop(context),
                      ),
                    ],
                  ),
                  const SizedBox(
                    height: 20,
                  ), // Replace with SizedBox(height: 20) if error
                  const Text(
                    "The Challenge:",
                    style: TextStyle(fontWeight: FontWeight.bold),
                  ),
                  Text(description, style: const TextStyle(fontSize: 15)),
                  const SizedBox(height: 20),
                  Row(
                    mainAxisAlignment: MainAxisAlignment.spaceBetween,
                    children: [
                      const Text(
                        "Anxiety Level:",
                        style: TextStyle(fontWeight: FontWeight.bold),
                      ),
                      Text(
                        "${entry['anxiety']}/10",
                        style: TextStyle(
                          color: Theme.of(context).colorScheme.primary,
                          fontWeight: FontWeight.bold,
                        ),
                      ),
                    ],
                  ),
                  const SizedBox(height: 20),
                  const Text(
                    "Your Reflection:",
                    style: TextStyle(fontWeight: FontWeight.bold),
                  ),
                  const SizedBox(height: 5),
                  Text(
                    entry['note'] ?? "No notes added",
                    style: const TextStyle(
                      fontStyle: FontStyle.italic,
                      color: Colors.grey,
                    ),
                  ),
                  const SizedBox(height: 20),
                  Text(
                    "Date: ${entry['date']}",
                    style: const TextStyle(fontSize: 12, color: Colors.grey),
                  ),
                  const SizedBox(height: 20),
                  SizedBox(
                    width: double.infinity,
                    child: ElevatedButton(
                      onPressed: () => Navigator.pop(context),
                      style: ElevatedButton.styleFrom(
                        backgroundColor: Theme.of(context).colorScheme.primary,
                        foregroundColor: Colors.white,
                        shape: RoundedRectangleBorder(
                          borderRadius: BorderRadius.circular(15),
                        ),
                      ),
                      child: const Text("Close"),
                    ),
                  ),
                ],
              ),
            ),
          ),
        );
      },
      transitionBuilder: (context, anim1, anim2, child) {
        return FadeTransition(
          opacity: anim1,
          child: ScaleTransition(
            scale: Tween<double>(begin: 0.8, end: 1.0).animate(anim1),
            child: SlideTransition(
              position:
                  Tween<Offset>(
                    begin: const Offset(0, 0.2),
                    end: Offset.zero,
                  ).animate(
                    CurvedAnimation(parent: anim1, curve: Curves.easeOutBack),
                  ),
              child: child,
            ),
          ),
        );
      },
    );
  }

  // 3. The actual build method (The missing piece!)
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text("Growth Journey"),
        elevation: 0,
        // backgroundColor: Colors.transparent,
      ),
      body: FutureBuilder<List<Map<String, dynamic>>>(
        future: _loadHistory(),
        builder: (context, snapshot) {
          if (snapshot.connectionState == ConnectionState.waiting)
            return theLoadingScreen();
          if (!snapshot.hasData || snapshot.data!.isEmpty) {
            return Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  const Icon(Icons.auto_stories, size: 80, color: Colors.grey),
                  const SizedBox(height: 20),
                  Text(
                    "No reflections yet. Start your journey!",
                    style: const TextStyle(color: Colors.grey),
                  ),
                ],
              ),
            );
          }

          List<Map<String, dynamic>> history = snapshot.data!;
          return ListView.builder(
            padding: const EdgeInsets.all(24.0),
            physics: const BouncingScrollPhysics(),
            itemCount: history.length,
            itemBuilder: (context, index) {
              var entry = history[index];
              return Container(
                margin: const EdgeInsets.only(bottom: 16),
                decoration: BoxDecoration(
                  color: Theme.of(context).colorScheme.surfaceContainerHighest,
                  borderRadius: BorderRadius.circular(20),
                ),
                child: Material(
                  color: Colors.transparent,
                  child: ListTile(
                    contentPadding: const EdgeInsets.all(16),
                    title: Text(
                      entry['title'] ?? "Challenge",
                      style: const TextStyle(fontWeight: FontWeight.bold),
                    ),
                    subtitle: Column(
                      crossAxisAlignment: CrossAxisAlignment.start,
                      children: [
                        const SizedBox(height: 8),
                        Text(
                          entry['note'] ?? "No notes added",
                          style: const TextStyle(fontStyle: FontStyle.italic),
                          maxLines: 2,
                          overflow: TextOverflow.ellipsis,
                        ),
                        const SizedBox(height: 10),
                        Row(
                          mainAxisAlignment: MainAxisAlignment.spaceBetween,
                          children: [
                            Text(
                              "Anxiety: ${entry['anxiety']}/10",
                              style: TextStyle(
                                fontWeight: FontWeight.bold,
                                color: Theme.of(context).colorScheme.primary,
                              ),
                            ),
                            Text(
                              entry['date'] ?? "",
                              style: const TextStyle(
                                fontSize: 12,
                                color: Colors.grey,
                              ),
                            ),
                          ],
                        ),
                      ],
                    ),
                    trailing: const Icon(Icons.chevron_right),
                    onTap: () => _showTaskDetail(context, entry),
                  ),
                ),
              );
            },
          );
        },
      ),
    );
  }
}

// --- FAQ SCREEN ---
class FAQScreen extends StatelessWidget {
  const FAQScreen({super.key});

  // 1. DEFINE FAQ DATA USING TRANSLATION KEYS
  final List<Map<String, String>> _faqKeys = const [
    {"q": "faq_q1", "a": "faq_a1"},
    {"q": "faq_q2", "a": "faq_a2"},
    {"q": "faq_q3", "a": "faq_a3"},
    {"q": "faq_q4", "a": "faq_a4"},
    {"q": "faq_q5", "a": "faq_a5"},
    {"q": "faq_q6", "a": "faq_a6"},
    {"q": "faq_q7", "a": "faq_a7"},
    {"q": "faq_q8", "a": "faq_a8"},
    {"q": "faq_q9", "a": "faq_a9"},
  ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Tr('help_faq'), elevation: 0),
      // LISTEN TO LANGUAGE FOR INSTANT SWITCHING
      body: ValueListenableBuilder<String>(
        valueListenable: GlobalSettings.language,
        builder: (context, lang, _) {
          return SingleChildScrollView(
            padding: const EdgeInsets.all(24.0),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Tr(
                  'common_questions',
                  style: const TextStyle(
                    fontSize: 24,
                    fontWeight: FontWeight.bold,
                  ),
                ),
                const SizedBox(height: 20),
                // MAP OVER KEYS, USE Tr() FOR TITLE AND CONTENT
                ..._faqKeys.map((faq) => _buildFAQItem(context, faq)),
                const SizedBox(height: 40),
                Center(
                  child: Tr(
                    'keep_blooming',
                    style: TextStyle(
                      fontStyle: FontStyle.italic,
                      color: Colors.grey[600],
                    ),
                  ),
                ),
              ],
            ),
          );
        },
      ),
    );
  }

  // 2. HELPER METHOD WITH EXPLICIT COLORS & CLIPRRECT (FIXES INK SPLASH WARNING & CORNERS)
  Widget _buildFAQItem(BuildContext context, Map<String, String> faq) {
    final bgColor = Theme.of(context).colorScheme.surfaceContainerHighest;
    final radius = 15.0;

    return Container(
      margin: const EdgeInsets.only(bottom: 12),
      // ClipRRect forces children (expanded content) to stay inside rounded corners
      child: ClipRRect(
        borderRadius: BorderRadius.circular(radius),
        child: ExpansionTile(
          // EXPLICIT COLORS FIX THE "INK SPLASH INVISIBLE" WARNING
          backgroundColor: bgColor, // Color when OPEN
          collapsedBackgroundColor: bgColor, // Color when CLOSED
          // Ensure ink splash shape matches the rounded container
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(radius),
          ),
          collapsedShape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(radius),
          ),

          title: Tr(
            faq['q']!,
            style: const TextStyle(fontWeight: FontWeight.w600),
          ),
          children: [
            Padding(
              padding: const EdgeInsets.all(16.0),
              child: Tr(
                faq['a']!,
                style: TextStyle(
                  fontSize: 15,
                  color: Theme.of(context).colorScheme.onSurfaceVariant,
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
}

// --- MILESTONE SCREEN (FINAL: INSTANT SYNC, GAUGE-STYLE, FIRST TILE LOCKED) ---
class MilestoneScreen extends StatelessWidget {
  final User user;
  const MilestoneScreen({super.key, required this.user});

  @override
  Widget build(BuildContext context) {
    // LISTEN TO GLOBAL SETTINGS (INSTANT LOCAL STATE) - IDENTICAL TO GAUGE
    return ValueListenableBuilder<Map<String, dynamic>>(
      valueListenable: GlobalSettings.userProfile,
      builder: (context, profile, _) {
        // 1. CALCULATE PROGRESS (0.0 to 1.0) - SAME FORMULA AS GAUGE
        final double score =
            (profile['confidenceScore'] as num?)?.toDouble() ?? 0.0;
        final double progress = (score / MAX_CONFIDENCE_SCORE).clamp(0.0, 1.0);
        return ValueListenableBuilder<String>(
          valueListenable: GlobalSettings.language,
          builder: (context, lang, _) {
            return Scaffold(
              appBar: AppBar(
                title: Tr('your_growth_path'),
                elevation: 0,
                centerTitle: true,
              ),
              body: SingleChildScrollView(
                physics: const BouncingScrollPhysics(),
                child: LayoutBuilder(
                  builder: (context, constraints) {
                    // ----- LAYOUT CONSTANTS (MATCHES MILESTONE SPACING) -----
                    final double width = constraints.maxWidth;
                    const double nodeRadius = 36.0;
                    const double horizontalPadding = 24.0;
                    final double leftX = horizontalPadding + nodeRadius;
                    final double rightX =
                        width - horizontalPadding - nodeRadius;
                    const double startY = 80.0;
                    const double spacing =
                        150.0; // Visual Spacing matches Score % (20%, 40%, 60%, 100%)

                    // GENERATE NODES (Visual positions fixed)
                    final List<_NodeData> nodes = List.generate(
                      GAME_MILESTONES.length,
                      (index) {
                        final bool isLeft = index % 2 == 0;
                        return _NodeData(
                          index: index,
                          isLeft: isLeft,
                          center: Offset(
                            isLeft ? leftX : rightX,
                            80.0 + index * 150.0,
                          ),
                          radius: nodeRadius,
                        );
                      },
                    );

                    final double totalHeight =
                        80.0 + GAME_MILESTONES.length * 150.0 + 100.0;
                    final Size canvasSize = Size(
                      constraints.maxWidth,
                      totalHeight,
                    );

                    return SizedBox(
                      width: constraints.maxWidth,
                      height: totalHeight,
                      child: Stack(
                        clipBehavior: Clip.none,
                        children: [
                          // LAYER 1: FOREST DOODLES
                          CustomPaint(
                            size: canvasSize,
                            painter: _ForestDoodlePainter(
                              nodes: nodes,
                              color: GlobalSettings.themeColor.value,
                              screenSize: canvasSize,
                            ),
                          ),

                          // LAYER 2: PATH (SIMPLE PROGRESS PAINTER - GAUGE STYLE)
                          CustomPaint(
                            size: canvasSize,
                            painter: _MilestonePathPainter(
                              color: GlobalSettings.themeColor.value,
                              nodes: nodes,
                              progress: progress, // Live Progress Double
                            ),
                          ),

                          // LAYER 3: NODES
                          ...nodes.map((node) {
                            final m = GAME_MILESTONES[node.index];
                            final pos = node.center;
                            final r = node.radius;
                            // NODE UNLOCK LOGIC: Score >= RequiredScore (Milestone 0 needs 20 now)
                            final double currentScore =
                                (GlobalSettings
                                            .userProfile
                                            .value['confidenceScore']
                                        as num?)
                                    ?.toDouble() ??
                                0.0;
                            final bool reached =
                                currentScore >= m.requiredScore.toDouble();

                            return Positioned(
                              left: pos.dx - r,
                              top: pos.dy - r,
                              child: _StaticNodeWidget(
                                node: node,
                                milestone: m,
                                radius: r,
                                lang: lang,
                                screenWidth: constraints.maxWidth,
                                isReached: reached,
                              ),
                            );
                          }),
                        ],
                      ),
                    );
                  },
                ),
              ),
            );
          },
        );
      },
    );
  }
}

// ============================================================
// PRODUCTION MILESTONE PATH PAINTER (FIXED METRICS TO LIST)
// ============================================================
class _MilestonePathPainter extends CustomPainter {
  final Color color;
  final List<_NodeData> nodes;
  final double progress;

  _MilestonePathPainter({
    required this.color,
    required this.nodes,
    required this.progress,
  });

  @override
  void paint(Canvas canvas, Size size) {
    try {
      _paintInternal(canvas, size);
    } catch (e, st) {
      debugPrint("Path Painter Error: $e");
    }
  }

  void _paintInternal(Canvas canvas, Size size) {
    if (nodes.length < 2) return;

    // 1. Background Track (Neutral Grey)
    final bgPaint = Paint()
      ..color = Colors.grey.withOpacity(0.3)
      ..strokeWidth = 6
      ..style = PaintingStyle.stroke
      ..strokeCap = StrokeCap.round;

    // 2. Glow
    final glowPaint = Paint()
      ..color = color.withOpacity(0.4)
      ..strokeWidth = 16
      ..style = PaintingStyle.stroke
      ..strokeCap = StrokeCap.round
      ..maskFilter = const MaskFilter.blur(BlurStyle.normal, 10);

    // 3. Halo (White Border - VISIBILITY FIX)
    final haloPaint = Paint()
      ..color = Colors.white
      ..strokeWidth = 8
      ..style = PaintingStyle.stroke
      ..strokeCap = StrokeCap.round;

    // 4. Core Color (Theme)
    final fgPaint = Paint()
      ..color = color
      ..strokeWidth = 5
      ..style = PaintingStyle.stroke
      ..strokeCap = StrokeCap.round;

    // 1. Build Full Path Geometry
    final fullPath = Path();
    fullPath.moveTo(nodes.first.center.dx, nodes.first.center.dy);
    for (int i = 0; i < nodes.length - 1; i++) {
      _addSegment(fullPath, nodes[i].center, nodes[i + 1].center);
    }

    // 1. Draw Background Track
    canvas.drawPath(fullPath, bgPaint);

    // 2. Draw Foreground Progress
    // CRITICAL FIX: .toList() to allow multiple iterations
    final metrics = fullPath.computeMetrics().toList();

    if (metrics.isNotEmpty) {
      final totalLength = metrics.fold(0.0, (sum, m) => sum + m.length);
      final targetLength = totalLength * progress.clamp(0.0, 1.0);

      double drawn = 0.0;
      for (final metric in metrics) {
        final remaining = targetLength - drawn;
        if (remaining <= 0) break;

        if (remaining >= metric.length) {
          // Draw Full Segment (Glow -> Halo -> Core)
          final segmentPath = metric.extractPath(0, metric.length);
          canvas.drawPath(segmentPath, glowPaint);
          canvas.drawPath(segmentPath, haloPaint);
          canvas.drawPath(segmentPath, fgPaint);
        } else {
          // Draw Partial Segment (Gradual Fill)
          final partialPath = metric.extractPath(0, remaining);
          canvas.drawPath(partialPath, glowPaint);
          canvas.drawPath(partialPath, haloPaint);
          canvas.drawPath(partialPath, fgPaint);
        }
        drawn += metric.length;
      }
    }
  }

  void _addSegment(Path p, Offset p0, Offset p1) {
    if (p0 == p1) return;
    final cp1 = Offset((p0.dx + p1.dx) / 2, p0.dy + (p1.dy - p0.dy) * 0.4);
    final cp2 = Offset((p0.dx + p1.dx) / 2, p0.dy + (p1.dy - p0.dy) * 0.6);
    p.cubicTo(cp1.dx, cp1.dy, cp2.dx, cp2.dy, p1.dx, p1.dy);
  }

  @override
  bool shouldRepaint(covariant _MilestonePathPainter old) {
    return old.color != color || old.progress != progress || old.nodes != nodes;
  }
}

// ============================================================
// FOREST DOODLE PAINTER (SAFE, DENSE)
// ============================================================
class _ForestDoodlePainter extends CustomPainter {
  final List<_NodeData> nodes;
  final Color color;
  final Size screenSize;
  _ForestDoodlePainter({
    required this.nodes,
    required this.color,
    required this.screenSize,
  });
  @override
  void paint(Canvas canvas, Size size) {
    try {
      _paintInternal(canvas, size);
    } catch (_) {}
  }

  void _paintInternal(Canvas canvas, Size size) {
    if (nodes.length < 2) return;
    final random = Random(42);
    final decorationPaint = Paint()
      ..color = color.withOpacity(0.06)
      ..style = PaintingStyle.fill;
    final trunkPaint = Paint()
      ..color = Colors.brown.withOpacity(0.1)
      ..style = PaintingStyle.fill;
    final topPaint = Paint()
      ..color = color.withOpacity(0.07)
      ..style = PaintingStyle.fill;
    final grassPaint = Paint()
      ..color = color.withOpacity(0.1)
      ..style = PaintingStyle.fill;
    final rockPaint = Paint()
      ..color = Colors.grey.withOpacity(0.08)
      ..style = PaintingStyle.fill;
    final flowerPaint = Paint()
      ..color = Colors.pink.withOpacity(0.12)
      ..style = PaintingStyle.fill;

    // 1. Far Background
    for (int i = 0; i < 12; i++) {
      double x = Random().nextDouble() * size.width;
      double y = Random().nextDouble() * size.height * 0.8;
      double scale = 0.8 + Random().nextDouble() * 0.8;
      _drawPineTree(
        canvas,
        x,
        y,
        80 * scale,
        Paint()
          ..color = Colors.brown.withOpacity(0.1)
          ..style = PaintingStyle.fill,
        decorationPaint,
      );
    }

    // 2. Path Decorations
    for (int i = 0; i < nodes.length - 1; i++) {
      final p0 = nodes[i].center;
      final p1 = nodes[i + 1].center;
      if (p0 == p1) continue;
      double segLen = (p1 - p0).distance;
      int decorationCount = (segLen / 35).floor().clamp(2, 6);
      for (int d = 0; d < decorationCount; d++) {
        double t = (d + 1) / (decorationCount + 1);
        Offset pos = _getPointOnCurve(p0, p1, t);
        Offset tangent = _getTangentOnCurve(p0, p1, t);
        double angle = atan2(tangent.dy, tangent.dx);
        double perpAngle = angle + (Random().nextBool() ? -pi / 2 : pi / 2);
        double dist = 30 + Random().nextDouble() * 70;
        double drawX = pos.dx + cos(perpAngle) * dist;
        double drawY = pos.dy + sin(perpAngle) * dist;
        if (drawX < 20 ||
            drawX > size.width - 20 ||
            drawY < 20 ||
            drawY > size.height - 20)
          continue;
        double scale = 0.5 + Random().nextDouble() * 1.0;
        int type = Random().nextInt(10);
        if (type < 3)
          _drawPineTree(
            canvas,
            drawX,
            drawY,
            40 * scale,
            Paint()
              ..color = Colors.brown.withOpacity(0.1)
              ..style = PaintingStyle.fill,
            decorationPaint,
          );
        else if (type < 5)
          _drawBush(canvas, drawX, drawY, 20 * scale, decorationPaint);
        else if (type < 7)
          _drawGrassTuft(canvas, drawX, drawY, 15 * scale, decorationPaint);
        else if (type < 9)
          _drawRock(
            canvas,
            drawX,
            drawY,
            10 * scale,
            Paint()
              ..color = Colors.grey.withOpacity(0.08)
              ..style = PaintingStyle.fill,
          );
        else
          _drawFlower(
            canvas,
            drawX,
            drawY,
            8 * scale,
            Paint()
              ..color = Colors.pink.withOpacity(0.12)
              ..style = PaintingStyle.fill,
          );
      }
    }

    // 3. Node Clusters
    for (var node in nodes) {
      int clusterCount = 5 + Random().nextInt(3);
      for (int i = 0; i < clusterCount; i++) {
        double angle =
            (i / clusterCount) * 2 * pi + Random().nextDouble() * 0.5;
        double dist = node.radius + 15 + Random().nextDouble() * 20;
        double fx = node.center.dx + cos(angle) * dist;
        double fy = node.center.dy + sin(angle) * dist;
        if (fx < 15 || fx > size.width - 15 || fy < 15 || fy > size.height - 15)
          continue;
        double scale = 0.4 + Random().nextDouble() * 0.6;
        int type = Random().nextInt(5);
        if (type == 0)
          _drawFlower(
            canvas,
            fx,
            fy,
            6 * scale,
            Paint()
              ..color = Colors.pink.withOpacity(0.12)
              ..style = PaintingStyle.fill,
          );
        else if (type == 1)
          _drawMushroom(canvas, fx, fy, 5 * scale, decorationPaint);
        else if (type == 2)
          _drawGrassTuft(canvas, fx, fy, 10 * scale, decorationPaint);
        else if (type == 3)
          _drawRock(
            canvas,
            fx,
            fy,
            6 * scale,
            Paint()
              ..color = Colors.grey.withOpacity(0.08)
              ..style = PaintingStyle.fill,
          );
        else
          _drawSmallFern(canvas, fx, fy, 8 * scale, decorationPaint);
      }
    }
  }

  Offset _getPointOnCurve(Offset p0, Offset p1, double t) {
    final cp1 = Offset((p0.dx + p1.dx) / 2, p0.dy + (p1.dy - p0.dy) * 0.4);
    final cp2 = Offset((p0.dx + p1.dx) / 2, p0.dy + (p1.dy - p0.dy) * 0.6);
    double u = 1 - t, uu = u * u, uuu = uu * u, tt = t * t, ttt = t * t * t;
    return p0 * uuu + cp1 * (3 * uu * t) + cp2 * (3 * u * tt) + p1 * ttt;
  }

  Offset _getTangentOnCurve(Offset p0, Offset p1, double t) {
    final cp1 = Offset((p0.dx + p1.dx) / 2, p0.dy + (p1.dy - p0.dy) * 0.4);
    final cp2 = Offset((p0.dx + p1.dx) / 2, p0.dy + (p1.dy - p0.dy) * 0.6);
    double u = 1 - t;
    return (cp1 - p0) * (3 * u * u) +
        (cp2 - cp1) * (6 * u * t) +
        (p1 - cp2) * (3 * t * t);
  }

  void _drawBush(Canvas canvas, double x, double y, double r, Paint paint) {
    canvas.drawCircle(Offset(x, y), r, paint);
    canvas.drawCircle(Offset(x - r * 0.6, y + r * 0.3), r * 0.7, paint);
    canvas.drawCircle(Offset(x + r * 0.6, y + r * 0.3), r * 0.7, paint);
  }

  void _drawPineTree(
    Canvas canvas,
    double x,
    double y,
    double h,
    Paint trunkPaint,
    Paint topPaint,
  ) {
    double tw = 2.0, th = h * 0.2;
    canvas.drawRect(
      Rect.fromCenter(center: Offset(x, y + h * 0.35), width: tw, height: th),
      trunkPaint,
    );
    Path p = Path();
    double cx = x;
    p.moveTo(cx, y + h * 0.2);
    p.lineTo(cx - h * 0.3, y + h * 0.5);
    p.lineTo(cx + h * 0.3, y + h * 0.5);
    p.close();
    p.moveTo(cx, y - h * 0.05);
    p.lineTo(cx - h * 0.2, y + h * 0.2);
    p.lineTo(cx + h * 0.2, y + h * 0.2);
    p.close();
    p.moveTo(cx, y - h * 0.3);
    p.lineTo(cx - h * 0.1, y - h * 0.05);
    p.lineTo(cx + h * 0.1, y - h * 0.05);
    p.close();
    canvas.drawPath(p, topPaint);
  }

  void _drawFlower(
    Canvas canvas,
    double x,
    double y,
    double size,
    Paint paint,
  ) {
    for (int i = 0; i < 5; i++) {
      double a = i * 2 * pi / 5;
      canvas.drawCircle(
        Offset(x + cos(a) * size * 0.7, y + sin(a) * size * 0.7),
        size * 0.45,
        paint,
      );
    }
    canvas.drawCircle(
      Offset(x, y),
      size * 0.35,
      Paint()..color = Colors.yellow.withOpacity(0.9),
    );
  }

  void _drawMushroom(
    Canvas canvas,
    double x,
    double y,
    double size,
    Paint paint,
  ) {
    Path cap = Path();
    cap.moveTo(x - size, y);
    cap.quadraticBezierTo(x, y - size * 1.5, x + size, y);
    cap.close();
    canvas.drawPath(cap, Paint()..color = Colors.redAccent.withOpacity(0.6));
    canvas.drawCircle(
      Offset(x - size * 0.3, y - size * 0.3),
      size * 0.15,
      Paint()..color = Colors.white.withOpacity(0.8),
    );
    canvas.drawRect(
      Rect.fromCenter(
        center: Offset(x, y + size * 0.6),
        width: size * 0.3,
        height: size,
      ),
      Paint()..color = Colors.white.withOpacity(0.5),
    );
  }

  void _drawGrassTuft(
    Canvas canvas,
    double x,
    double y,
    double h,
    Paint paint,
  ) {
    Path p = Path();
    p.moveTo(x, y);
    for (int i = 0; i < 3; i++) {
      double bladeAngle = -pi / 2 + (i - 1) * 0.4;
      p.moveTo(x, y);
      p.quadraticBezierTo(
        x + cos(bladeAngle) * h / 2,
        y - h / 2,
        x + cos(bladeAngle) * h / 3,
        y - h,
      );
    }
    canvas.drawPath(p, paint);
  }

  void _drawRock(Canvas canvas, double x, double y, double size, Paint paint) {
    Path p = Path();
    for (int i = 0; i < 6; i++) {
      double a = i * 2 * pi / 6;
      double r = size * (0.8 + (i % 2) * 0.4);
      if (i == 0)
        p.moveTo(x + cos(a) * r, y + sin(a) * r);
      else
        p.lineTo(x + cos(a) * r, y + sin(a) * r);
    }
    p.close();
    canvas.drawPath(p, paint);
  }

  void _drawSmallFern(
    Canvas canvas,
    double x,
    double y,
    double h,
    Paint paint,
  ) {
    Path p = Path();
    p.moveTo(x, y);
    p.quadraticBezierTo(x - h * 0.5, y - h * 0.3, x - h * 0.3, y - h);
    p.moveTo(x, y);
    p.quadraticBezierTo(x + h * 0.5, y - h * 0.3, x + h * 0.3, y - h);
    canvas.drawPath(p, paint);
  }

  @override
  bool shouldRepaint(covariant _ForestDoodlePainter old) =>
      old.nodes != nodes || old.color != color || old.screenSize != screenSize;
}

// ============================================================
// STATIC NODE WIDGET & DATA
// ============================================================
class _StaticNodeWidget extends StatelessWidget {
  final _NodeData node;
  final MilestoneDefinition milestone;
  final double radius;
  final String lang;
  final double screenWidth;
  final bool isReached;
  const _StaticNodeWidget({
    required this.node,
    required this.milestone,
    required this.radius,
    required this.lang,
    required this.screenWidth,
    required this.isReached,
  });
  @override
  Widget build(BuildContext context) {
    final c = isReached ? milestone.color : Colors.grey.shade400;
    final bg = isReached
        ? milestone.color.withOpacity(0.15)
        : Colors.grey.shade100;
    final darkBg = isReached
        ? milestone.color.withOpacity(0.2)
        : Colors.grey.shade800;
    final isLeft = node.isLeft;
    final align = isLeft ? TextAlign.right : TextAlign.left;
    final cross = isLeft ? CrossAxisAlignment.end : CrossAxisAlignment.start;
    double gap = 16.0;
    double avail = isLeft
        ? node.center.dx - radius - gap
        : screenWidth - (node.center.dx + radius) - gap;
    double labelWidth = avail.clamp(100.0, 200.0);
    return Column(
      mainAxisSize: MainAxisSize.min,
      crossAxisAlignment: cross,
      children: [
        AnimatedContainer(
          duration: const Duration(milliseconds: 600),
          curve: Curves.easeOutBack,
          width: radius * 2,
          height: radius * 2,
          decoration: BoxDecoration(
            shape: BoxShape.circle,
            color: Theme.of(context).brightness == Brightness.dark
                ? darkBg
                : bg,
            border: Border.all(color: c, width: 3.5),
            boxShadow: isReached
                ? [
                    BoxShadow(
                      color: c.withOpacity(0.4),
                      blurRadius: 12,
                      spreadRadius: 2,
                    ),
                  ]
                : [],
          ),
          child: Icon(
            //
            milestone.icon,
            color: isReached ? c : Colors.grey.shade400,
            size: radius * 0.65,
          ),
        ),
        const SizedBox(height: 10),
        SizedBox(
          width: labelWidth,
          child: Column(
            crossAxisAlignment: cross,
            children: [
              Tr(
                milestone.titleKey,
                style: TextStyle(
                  fontWeight: FontWeight.bold,
                  fontSize: 13,
                  color: isReached ? Colors.black87 : Colors.grey,
                ),
                textAlign: align,
                maxLines: 2,
                overflow: TextOverflow.ellipsis,
              ),
              Tr(
                milestone.rewardKey,
                style: TextStyle(fontSize: 11, color: Colors.grey.shade600),
                textAlign: align,
                maxLines: 1,
                overflow: TextOverflow.ellipsis,
              ),
              if (!isReached)
                Padding(
                  padding: const EdgeInsets.only(top: 8.0),
                  child: Text(
                    AppTexts.get(
                      'milestone_need_score',
                      lang,
                    ).replaceAll('{0}', '${milestone.requiredScore}'),
                    style: TextStyle(fontSize: 11, color: Colors.grey.shade500),
                    textAlign: align,
                  ),
                )
              else
                Padding(
                  padding: const EdgeInsets.only(top: 8.0),
                  child: Container(
                    padding: const EdgeInsets.symmetric(
                      horizontal: 10,
                      vertical: 4,
                    ),
                    decoration: BoxDecoration(
                      color: Colors.green.withOpacity(0.1),
                      borderRadius: BorderRadius.circular(12),
                    ),
                    child: Row(
                      mainAxisSize: MainAxisSize.min,
                      children: [
                        Icon(
                          Icons.check_circle,
                          size: 13,
                          color: Colors.green.shade700,
                        ),
                        const SizedBox(width: 4),
                        Text(
                          "Reached",
                          style: TextStyle(
                            fontSize: 11,
                            fontWeight: FontWeight.bold,
                            color: Colors.green.shade700,
                          ),
                        ),
                      ],
                    ),
                  ),
                ),
            ],
          ),
        ),
      ],
    );
  }
}

class _NodeData {
  final int index;
  final bool isLeft;
  final Offset center;
  final double radius;
  _NodeData({
    required this.index,
    required this.isLeft,
    required this.center,
    required this.radius,
  });
  @override
  bool operator ==(Object o) =>
      identical(this, o) ||
      o is _NodeData &&
          index == o.index &&
          isLeft == o.isLeft &&
          center == o.center &&
          radius == o.radius;
  @override
  int get hashCode =>
      index.hashCode ^ isLeft.hashCode ^ center.hashCode ^ radius.hashCode;
}

Widget buildBloomIcon(IconData icon, Color color, {double size = 28}) {
  return Container(
    padding: const EdgeInsets.all(10),
    decoration: BoxDecoration(
      // This creates a soft, glowing background for the icon
      gradient: RadialGradient(
        colors: [color.withOpacity(0.3), color.withOpacity(0.05)],
      ),
      shape: BoxShape.circle,
      border: Border.all(color: color.withOpacity(0.2), width: 1),
    ),
    child: Icon(icon, color: color, size: size),
  );
}

class ShopScreen extends StatefulWidget {
  final User user;
  const ShopScreen({super.key, required this.user});

  @override
  State<ShopScreen> createState() => _ShopScreenState();
}

class _ShopScreenState extends State<ShopScreen> {
  int _points = 0;
  int _ownedFreezes = 0;
  int _activeFreezes = 0;
  bool _isLoading = false;
  final int freezeCost = 50; // Cost of one freeze

  @override
  void initState() {
    super.initState();
    _loadData();
  }

  Future<void> _loadData() async {
    var data = await UserService().getUserData(widget.user);
    setState(() {
      _points = data['totalPoints'] ?? 0;
      _ownedFreezes = data['streakFreezes'] ?? 0;
      _activeFreezes = data['activeFreezes'] ?? 0;
    });
  }

  void _buyFreeze() async {
    bool success = await UserService().buyStreakFreeze(widget.user, freezeCost);
    if (success) {
      _loadData();
      if (mounted)
        ScaffoldMessenger.of(
          context,
        ).showSnackBar(SnackBar(content: Tr('freeze_purchased')));
    } else {
      if (mounted)
        ScaffoldMessenger.of(
          context,
        ).showSnackBar(SnackBar(content: Tr('not_enough_points')));
    }
  }

  Future<void> _equipFreeze() async {
    setState(() => _isLoading = true);
    try {
      await UserService().equipFreeze(widget.user);
      // ✅ Transaction succeeded – refresh data and show success
      await _loadData();
      if (mounted)
        ScaffoldMessenger.of(
          context,
        ).showSnackBar(SnackBar(content: Tr('freeze_equipped')));
    } catch (e) {
      // ❌ Transaction failed (condition not met or other error)
      if (mounted)
        ScaffoldMessenger.of(
          context,
        ).showSnackBar(SnackBar(content: Text('Error: $e')));
    } finally {
      // ✅ Always reset the loading flag, even when an error occurs
      if (mounted) setState(() => _isLoading = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Tr('shop_title'), elevation: 0), // Tr Widget
      // LISTEN TO LANGUAGE CHANGES
      body: ValueListenableBuilder<String>(
        valueListenable: GlobalSettings.language,
        builder: (context, lang, _) {
          return Padding(
            padding: const EdgeInsets.all(24.0),
            child: Column(
              children: [
                // Points Card
                Container(
                  padding: const EdgeInsets.all(20),
                  decoration: BoxDecoration(
                    color: Theme.of(context).colorScheme.primaryContainer,
                    borderRadius: BorderRadius.circular(20),
                  ),
                  child: Row(
                    mainAxisAlignment: MainAxisAlignment.spaceBetween,
                    children: [
                      Tr(
                        'your_points',
                        style: const TextStyle(
                          fontSize: 18,
                          fontWeight: FontWeight.bold,
                        ),
                      ), // Widget
                      Text(
                        "$_points pts",
                        style: TextStyle(
                          fontSize: 22,
                          fontWeight: FontWeight.bold,
                          color: Theme.of(context).colorScheme.primary,
                        ),
                      ),
                    ],
                  ),
                ),
                const SizedBox(height: 30),
                Tr(
                  'available_items',
                  style: const TextStyle(
                    fontSize: 20,
                    fontWeight: FontWeight.bold,
                  ),
                ), // Widget
                const SizedBox(height: 20),

                // Streak Freeze Item
                Container(
                  padding: const EdgeInsets.all(20),
                  decoration: BoxDecoration(
                    color: Theme.of(
                      context,
                    ).colorScheme.surfaceContainerHighest,
                    borderRadius: BorderRadius.circular(25),
                    border: Border.all(
                      color: Theme.of(
                        context,
                      ).colorScheme.primaryContainer.withOpacity(0.5),
                    ),
                  ),
                  child: Row(
                    children: [
                      const Icon(Icons.ac_unit, size: 40, color: Colors.blue),
                      const SizedBox(width: 15),
                      Expanded(
                        child: Column(
                          crossAxisAlignment: CrossAxisAlignment.start,
                          children: [
                            Tr(
                              'streak_freeze',
                              style: const TextStyle(
                                fontSize: 18,
                                fontWeight: FontWeight.bold,
                              ),
                            ), // Widget
                            Text(
                              AppTexts.get('protects_streak', lang),
                              style: const TextStyle(
                                fontSize: 12,
                                color: Colors.grey,
                              ),
                            ), // String
                          ],
                        ),
                      ),
                      Column(
                        children: [
                          Text(
                            "$freezeCost pts",
                            style: TextStyle(
                              fontWeight: FontWeight.bold,
                              color: Theme.of(context).colorScheme.primary,
                            ),
                          ),
                          const SizedBox(height: 10),
                          ElevatedButton(
                            onPressed: _points >= freezeCost
                                ? _buyFreeze
                                : null,
                            child: Tr('buy'), // Widget
                          ),
                        ],
                      ),
                    ],
                  ),
                ),
                const SizedBox(height: 30),
                const Divider(),
                const SizedBox(height: 20),
                Tr(
                  'your_inventory',
                  style: const TextStyle(
                    fontSize: 18,
                    fontWeight: FontWeight.bold,
                  ),
                ), // Widget
                const SizedBox(height: 15),

                // Inventory ListTile
                ListTile(
                  leading: const Icon(
                    Icons.ac_unit_outlined,
                    color: Colors.blue,
                  ),
                  title: Tr('streak_freeze'), // Widget
                  subtitle: Text(
                    AppTexts.get('owned', lang) +
                        ': $_ownedFreezes | ' +
                        AppTexts.get('equipped', lang) +
                        ': $_activeFreezes/2',
                  ), // String
                  trailing: _activeFreezes < 2 && _ownedFreezes > 0
                      ? ElevatedButton(
                          onPressed: _isLoading ? null : _equipFreeze,
                          child: _isLoading
                              ? const SizedBox(
                                  width: 16,
                                  height: 16,
                                  child: CircularProgressIndicator(
                                    strokeWidth: 2,
                                  ),
                                )
                              : Tr('equip_freeze'), // Widget
                        )
                      : const SizedBox(),
                ),
              ],
            ),
          );
        },
      ),
    );
  }
}
